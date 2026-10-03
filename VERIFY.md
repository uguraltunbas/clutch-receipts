# How to check the receipts

Everything below works on a copy of this repository
(`git clone https://github.com/uguraltunbas/clutch-receipts`) with tools
that come with macOS, Linux or Windows. Nothing here needs our servers.

## 1. One file's fingerprint

```sh
shasum -a 256 2026-27/2026-10-20/seal-182205Z.json
```

(Linux: `sha256sum <file>`; Windows: `certutil -hashfile <file> SHA256`.)

The result must be the `sha256` on that file's line in `index.jsonl`, and the
`prev_sha256` written inside the next receipt.

## 2. When it arrived: GitHub's clock

Open the repository's **Activity** view:
<https://github.com/uguraltunbas/clutch-receipts/activity>. It lists every
push with the time GitHub received it. That time is GitHub's own record, so
a seal that arrived before a game's tip-off was written before the tip-off.

The dates shown next to commits elsewhere are written by whoever makes the
commit, so on their own they prove nothing — use the Activity view. It also
lists force pushes: history that was rewritten would show there.

## 3. The whole chain (Python 3, no packages)

Run this in the repository folder:

```python
import hashlib
import json

prev = None
for line in open("index.jsonl", encoding="ascii"):
    entry = json.loads(line)
    data = open(entry["path"], "rb").read()
    receipt = json.loads(data)
    assert hashlib.sha256(data).hexdigest() == entry["sha256"], entry["path"] + ": fingerprint"
    assert receipt["prev_sha256"] == entry["prev_sha256"] == prev, entry["path"] + ": chain"
    prev = entry["sha256"]
print("chain ok, latest", prev)
```

Every receipt names the fingerprint of the one before it. Changing any old
file changes its fingerprint, and from there on nothing matches.

## 4. A reveal against its seals

Before tip a seal carries, per game, `pro_commitment`: the SHA-256 of the
Pro-only numbers and a random secret (the salt). After the final the reveal
publishes both; recompute the fingerprint and look for it in the seals:

```python
import glob
import hashlib
import json


def canonical(obj):
    return json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=True).encode("ascii")


def fingerprint(game_id, opening):
    return hashlib.sha256(canonical({
        "format": "clutch-commitment-v1",
        "game_id": game_id,
        "prediction_id": opening["prediction_id"],
        "pro": opening["pro"],
        "salt": opening["salt"],
    })).hexdigest()


sealed = {}
for path in glob.glob("*/*/seal-*.json"):
    for game in json.load(open(path))["games"]:
        sealed.setdefault(game["prediction_id"], set()).add(game["pro_commitment"])

for path in sorted(glob.glob("*/*/reveal-*.json")):
    for game in json.load(open(path))["games"]:
        for opening in game["openings"]:
            found = fingerprint(game["game_id"], opening) in sealed.get(opening["prediction_id"], set())
            print(path, opening["prediction_id"], "matches its seal" if found else "is in no seal")
```

A different number, or a different salt, gives a different fingerprint — so
the numbers revealed are the numbers sealed.

## 5. Which seal a game is graded on

The app grades the last number published before tip. So for a game: take
the seals that arrived before its tip-off (section 2) and, of those, the
latest one that lists the game. Its `home_win_prob` is the graded number,
and its `prediction_id` is the `graded_prediction_id` in the reveal. If the
number changed after the last seal (a push that failed, say), the app shows
the game as not sealed rather than pretending it was.

## 6. The fields

A seal: `format`, `kind` (`"seal"`), `slate_date` (US Eastern), `season`,
`sealed_at`, `prev_sha256`, `n_games`, and per game in `games`: `game_id`,
`home`, `away`, `tip_utc`, `prediction_id`, `model_version`,
`home_win_prob`, `pro_commitment`.

A reveal: the same header with `revealed_at`, and per game: `game_id`,
`home`, `away`, `tip_utc`, `home_score`, `away_score`, `graded` (whether the
Ledger counts the game — preseason never counts), `graded_prediction_id`,
`home_win_prob`, `pick`, `correct`, and `openings` — for every prediction of
the game that was sealed: `prediction_id`, `home_win_prob`, `pro`
(`model_home_prob`, `market_home_prob`, `predicted_home_margin`,
`predicted_total`), `salt`, `pro_commitment`.

`home_win_prob` is the home team's chance to win. The pick is the home team
when it is 0.5 or more. The projected score is (total ± margin) / 2.

## 7. The exact form of every file (for programmers)

Each file is written in one exact form, so any language can rebuild the
bytes and get the same fingerprint:

- JSON in UTF-8; every string is printable ASCII (space to `~`).
- Object keys sorted by character code; no whitespace at all; `,` between
  items and `:` between a key and its value.
- Numbers are whole numbers only (scores, counts). Fractional values are
  strings with fixed decimals: probabilities 4 (`"0.6120"`), margins and
  totals 2 (`"-3.50"`). A missing value is `null`.
- A receipt file is exactly these bytes, with no newline at the end.
  `index.jsonl` holds one such object per line, each followed by a newline.
- `pro_commitment` is the SHA-256, in lowercase hex, of the canonical bytes
  of `{"format": "clutch-commitment-v1", "game_id", "prediction_id", "pro",
  "salt"}`.

The same rule in JavaScript (Node 18 or later, or any modern browser):

```js
function canonical(value) {
  if (value === null || typeof value !== "object") return JSON.stringify(value);
  if (Array.isArray(value)) return "[" + value.map(canonical).join(",") + "]";
  return "{" + Object.keys(value).sort()
    .map((key) => JSON.stringify(key) + ":" + canonical(value[key])).join(",") + "}";
}

function commitmentPreimage(gameId, opening) {
  return canonical({
    format: "clutch-commitment-v1",
    game_id: gameId,
    prediction_id: opening.prediction_id,
    pro: opening.pro,
    salt: opening.salt,
  });
}
```

And checking one file with Node (`node check.js <file>`):

```js
const crypto = require("crypto");
const fs = require("fs");

const sha256 = (bytes) => crypto.createHash("sha256").update(bytes).digest("hex");
const raw = fs.readFileSync(process.argv[2]);
const receipt = JSON.parse(raw.toString("utf8"));
console.log(canonical(receipt) === raw.toString("utf8") ? "canonical" : "NOT canonical", sha256(raw));
```
