# Clutch Receipts

Every win probability the Clutch app grades is published here **before its
game tips off**, in files that cannot be quietly changed afterwards.

Track records usually leave out the misses. This one can be checked by
anyone, without trusting us: the numbers are here, with GitHub's own clock
on every one, before the games are played. The app's Ledger grades exactly
these numbers — the wins and the misses.

## What is in this repository

| File | What it is |
| --- | --- |
| `<season>/<date>/seal-<time>Z.json` | A **seal**: every game of that night still to tip, with the number the app published for it. A new seal is added whenever a number changes during the day (new injury news, a new line), so the last seal before a tip-off holds the number that game is graded on. |
| `<season>/<date>/reveal-<time>Z.json` | A **reveal**, written after the games are final: the numbers that are only for Pro members before tip, which prove the seal's fingerprints, plus the final score and whether the call was right. |
| `index.jsonl` | One line per seal or reveal, oldest first, with each file's SHA-256 fingerprint. |
| `VERIFY.md` | How to check all of this yourself. |

Dates are the US Eastern date of the games; times are UTC.

## What a seal shows

For each game: the teams, the tip-off time, our home win probability (the
number the app shows everyone), the model version and the id of the
prediction. That is exactly what the free app shows.

The numbers that are only for Pro members before tip — our model's own
probability, the market's probability it used, the projected margin and
total — are sealed too, but as a fingerprint (`pro_commitment`): a SHA-256
of those numbers and a long random secret. The fingerprint gives nothing
away before the game. After the final the reveal publishes the numbers and
the secret, and anyone can recompute the fingerprint and see it matches the
one sealed before tip.

## Why you can trust it without trusting us

1. **GitHub's clock.** GitHub records when each push arrived. That time is
   GitHub's, not ours — see the repository's **Activity** view. A seal that
   arrived before tip-off was written before tip-off.
2. **The chain.** Every file names the SHA-256 of the one before it
   (`prev_sha256`). Changing any old file changes its fingerprint, and every
   later file stops matching. Rewriting history would also show up as a
   force push in the Activity view.
3. **The fingerprints.** The Pro-only numbers cannot be changed after the
   fact: a different number gives a different fingerprint.

Commits are authored as "Clutch Receipts <receipts@getclutch.app>" and
pushed by our scheduled jobs with a key that can write to this repository
only. The author line is a label; the proof is the chain and GitHub's clock.

## Check it yourself

`VERIFY.md` shows how, from a one-line `shasum` check to a full walk of the
chain in a few lines of Python or JavaScript.
