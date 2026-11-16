# Selected code excerpts

Short excerpts from the private codebase. Some are trimmed or lightly simplified for length (cuts are marked `// …`); the logic is unchanged. No configuration values or secrets are included.

---

## 1. Model output contract (Zod)

```ts
/** The analysis JSON the agent must return. This schema is the single source
 *  of truth; the agent prompt describes it in prose for the model. */
export const AnalysisResultSchema = z.object({
  reflection: z.string().min(1),
  themes: z.array(AnalysisThemeSchema).min(1).max(4),
  interpretation: z.string().min(1),
  deepQuestions: z.array(z.string().min(1)).min(2).max(3),
  shadowExercise: z.string().min(1),
  citations: z.array(AnalysisCitationSchema),
});

// In the API route, after the agent returns:
const parsed = AnalysisResultSchema.safeParse(parsedJson);
if (!parsed.success) {
  await safeRefund(); // the user doesn't pay for a malformed answer
  return NextResponse.json({ error: "Invalid analysis payload shape" }, { status: 502 });
}
```

**Why:** LLM output is untrusted input. Nothing reaches the database or the UI until it matches this exact shape. `safeParse` turns a schema failure into a handled 502 with a credit refund instead of an exception. The theme `category` is validated loosely on write (any non-empty string), so one off-taxonomy word from the model doesn't fail the whole analysis; the read path maps unknown categories to "uncategorized".

---

## 2. Compare-and-swap credit deduction

```ts
for (let attempt = 0; attempt < 4; attempt += 1) {
  const { subscription_credits: sub, bonus_credits: bonus } = await getUserCreditRow(userId);
  if (sub + bonus < cost) return { ok: false };

  // Spend subscription credits first, then bonus credits.
  const newSub = Math.max(0, sub - cost);
  const newBonus = bonus - Math.max(0, cost - sub);

  const { data, error } = await supabaseAdmin
    .from("users")
    .update({ subscription_credits: newSub, bonus_credits: newBonus })
    .eq("id", userId)
    .eq("subscription_credits", sub)   // only if nobody changed it
    .eq("bonus_credits", bonus)        // since we read it
    .select("id");

  if (error) throw error;
  if (data.length > 0) {
    return { ok: true, refund: { sub: sub - newSub, bonus: bonus - newBonus } };
  }
  // 0 rows updated: another request won the race. Re-read and retry.
}
return { ok: false };
```

**Why:** A plain read-then-write lets two concurrent analyses both see "1 credit left" and both run. The `WHERE` clause on the old values makes the update a compare-and-swap: only one request matches the row, and the other re-reads and retries. The function returns which pool each credit came from, so a refund after a failed run puts credits back in the right pool.

---

## 3. Idempotent Stripe webhook

```ts
/** Returns false if this id was already processed. */
export async function claimStripeSubscriptionCreditEvent(id: string) {
  const { error } = await supabaseAdmin
    .from("stripe_subscription_credit_events")
    .insert({ id }); // id is the primary key
  if (!error) return true;
  if (error.code === "23505") return false; // unique_violation: already claimed
  console.error("claimStripeSubscriptionCreditEvent:", error);
  return false;
}

// In the webhook handler (after signature verification):
case "invoice.paid": {
  // …
  const claimed = await claimStripeSubscriptionCreditEvent(`inv:${invoice.id}`);
  if (!claimed) break; // a Stripe retry; credits were already granted
  await refillSubscriptionCredits(userId, planCap);
  break;
}
```

**Why:** Stripe delivers webhooks at least once, so the same `invoice.paid` can arrive more than once. The database's unique constraint does the deduplication, so two retries arriving at the same moment can't both pass a "have I seen this?" check. On an unexpected error the handler doesn't grant credits; it fails closed.

---

## 4. Service-to-service auth with a timing-safe compare

```ts
// Hash both sides to fixed-length buffers so timingSafeEqual can compare
// values of different lengths without throwing.
const digest = (value: string) => createHash("sha256").update(value).digest();

const isValidInternalKey = (provided: string | null, expected: string) =>
  provided !== null && timingSafeEqual(digest(provided), digest(expected));

// In the agent's authenticate hook, before the user JWT is checked:
const internalKey = process.env.AGENT_INTERNAL_KEY?.trim();
if (!internalKey) {
  // Fail closed: a misconfigured server must not accept traffic.
  throw new HTTPException(500, { message: "Internal key not configured" });
}
if (!isValidInternalKey(request.headers.get("x-internal-key"), internalKey)) {
  throw new HTTPException(401, { message: "Missing or invalid internal service key" });
}
// …then verify the Supabase JWT and scope threads to that user.
```

**Why:** A user's JWT alone must not be enough to reach the agent. Credits, rate limits and usage metering live in the web app, so only the web server (which holds this key) may call it. `===` can stop at the first mismatching character, which leaks timing information; `timingSafeEqual` takes the same time either way, and hashing first gives both buffers the same length, which it requires.
