# Leadership 360

Scenario-based 360 built on the Five Levels of Leadership from *Developing the
Leader Within You 2.0* (Chapter 1), with each scenario drawn from a later
chapter's trait. Responses collect automatically — no codes to copy, paste,
or email around.

**Live:** https://jameszhang-maker.github.io/leadership-360/

## How it works

- **Leader** clicks "Assess myself," answers 15 scenarios, gets two links:
  a durable link to their own dashboard (`?leader=<id>`) and an invite link
  for raters (`?round=<id>&for=<name>`).
- **Raters** open the invite link, answer the same 15 scenarios about the
  leader, and submit — nothing to send back manually.
- **Report** (`?round=<id>&report=1`) shows a live, self-updating picture:
  one overview score across Maxwell's Five Levels, then a breakdown across
  nine named traits (one per book chapter), blind spots, hidden strengths,
  and the sharpest self-vs-rater disagreements.
- The leader's dashboard lists every round they've ever run, so this can be
  retaken every few months and compared over time. "Revise my
  self-assessment" updates the current round in place; "Start a new round"
  begins a fresh one under the same leader.

Nothing requires an account. Access control is entirely via long, unguessable
IDs in the URL — adequate for a pilot, not a substitute for real auth if this
ever handles higher-stakes feedback.

## Architecture

Static frontend (`index.html`, one file, no build step) on GitHub Pages,
talking to a small AWS backend:

- **API Gateway HTTP API** → **Lambda** (`backend/index.mjs`, Node 22,
  zero npm dependencies — the AWS SDK v3 ships in the managed runtime) →
  **DynamoDB** (`leadership360` table, single-table design, provisioned
  1 RCU/1 WCU — deliberately far under the Always-Free 25/25 ceiling).
- Lambda Function URLs were tried first (no API Gateway needed, avoids its
  12-months-only free tier) but every anonymous request returned 403 —
  confirmed to be an account-level guardrail on this AWS account, not a
  configuration mistake, by deleting and recreating the Function URL config
  and permission from scratch. API Gateway HTTP API is unaffected by that
  guardrail. Its free tier is 1M requests/month for 12 months, then
  $1/million after — at this app's pilot-scale usage (quarterly rounds, a
  handful of raters) that's fractions of a cent even outside the free tier.
- CORS is restricted to the GitHub Pages origin.
- `backend/` (Lambda source, IAM policy JSON, the provisioning script) is
  **not** published — see `.gitignore`. It contains the AWS account ID in
  policy ARNs; no reason to publish it even though none of it is a secret.

## Redeploying

**Frontend** — edit `index.html`, then:
```bash
git add index.html && git commit -m "..." && git push
```
GitHub Pages rebuilds automatically in under a minute.

**Backend** — edit `backend/index.mjs`, then re-run the provisioning script
(it's idempotent — safe to run any time, updates the Lambda code in place
and leaves everything else untouched):
```bash
./backend/setup-aws.sh
```

**Item bank** — the 15 scenarios live in the `ITEMS` array near the top of
`index.html`'s `<script>` block. Each item:

```js
{
  ch:[2],                 // source chapter(s) — drives the report's "next moves"
  w:{1:.7, 3:.3},         // Level weights (Maxwell's 5 levels) — overview only
  tw:{2:1},               // Trait weights (one per chapter) — the main breakdown
  self:"…",               // scenario stem, second person
  other:"…",               // same stem with {N} in place of "you/your"
  opts:[
    {t:"…", s:1},         // s = maturity score 1–5, never shown to the user
    …
  ]
}
```

Rules that matter when editing:
- Every option must be a **defensible real behaviour**, not a strawman —
  score 1 should be something a reasonable person would actually do, just
  less mature than score 5.
- Don't make score 5 the longest or most eloquent option, or raters will
  pick the "sounds best" option instead of the true one.
- `w` and `tw` each sum to 1 **within an item** (not across the whole bank)
  — level and trait scores are weighted means, not weighted sums.
- `ch` can list more than one chapter (used for `tw` splits and "next
  moves"); it's forgiving to get approximately right.

## Pilot protocol

1. Pick one leader to pilot with — someone genuinely curious about the gap
   between self-view and others'-view.
2. They assess themselves, then send the invite link to 4–6 raters: aim for
   their manager, 2–3 peers, and 2+ people they lead. Fewer than 4 makes the
   averages noisy; more than ~8 doesn't add much signal in a pilot.
3. Watch for: did raters actually finish (drop-off is the first sign the
   item count or wording needs cutting)? Did the report's blind-spot and
   "sharpest three" sections say something the leader didn't already know?
   Did any scenario feel unnatural or force a false choice?
4. Three months later, "Start a new round" and compare.

## Known limits (prototype, by design)

- No auth beyond unguessable IDs — anyone with a link can view or (for a
  rater link) submit to that round.
- No protection against a leader fabricating rater responses directly via
  the API — this is a trust instrument for a coaching relationship, not a
  locked-down survey tool.
- Fixed at 15 items / 5 levels / 9 traits / 5 options per scenario.
- DynamoDB is provisioned at 1 RCU/1 WCU on purpose (to guarantee free
  tier); a burst of many raters submitting in the same second could see a
  retried request rather than an instant one — not a failure, just a beat
  of latency.
