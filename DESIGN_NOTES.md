# Design Notes — KYC Rejection Assistant

## LLM vs. Deterministic Logic

The AI never decides *why* something failed — that's already known before the workflow even runs (expired ID, blurry photo, name mismatch — whatever reason comes in through the form). All the AI does is write the internal note and the customer message based on that known reason, following the format and tone set in its system prompt:

```
You are Voltify's KYC support assistant. You will be given a known rejection reason for a
failed identity verification. Output exactly two things:

1. INTERNAL NOTE — a single factual line for the support agent, no more than 15 words.
2. CUSTOMER MESSAGE — a short, clear, empathetic message asking the customer to fix the
   issue, in Voltify's brand voice (friendly but professional).

Never state a reason other than the one provided. Never guess additional details not given.
```

So the split is: **reason = deterministic, already known, fixed input** → **wording = the only thing the LLM generates**.

## Where the Human Checkpoint Sits

The check happens in Slack, after the AI drafts the message and before it reaches the customer. The supporter gets a Slack message with a button that opens an n8n form. In that form they can read the draft, edit it if needed, and hit submit — only then does the (approved or edited) message go out to the customer. Nothing the AI writes goes out on its own.

## Catching a Wrong or Hallucinated Detail

Because the internal note and customer message are short and tied to one known reason, the supporter reviewing the form can quickly check the draft against the actual rejection reason before approving it. If the AI ever stated something that wasn't in the original input, it would stand out immediately against that one known fact — the system prompt also explicitly tells it never to state a reason other than the one it was given, so any drift from that is easy to catch on review.

## Biggest Risk + Mitigation

**Risk:** if supporters don't review the drafts fast enough, customers wait longer than they should for a response.

**Mitigation:** email notifications alongside the Slack message, so a supporter isn't only relying on catching it in Slack — they get notified through a second channel too, reducing the chance a draft sits unreviewed.
