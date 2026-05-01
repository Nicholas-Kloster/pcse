[![Claude Code Friendly](https://img.shields.io/badge/Claude_Code-Friendly-blueviolet?logo=anthropic&logoColor=white)](https://claude.ai/code)

# PCSE — Portable Conversation State Embedding

Claude already remembers a lot about you.

- **Memory** — facts about you across all conversations
- **Projects** — persistent context, files, and instructions 
  within a defined scope
- **Conversation history** — what was said in prior sessions

What it doesn't remember is how you two work together.

Not what you talked about — but how. How compressed your 
communication is. How much needs to be explained. The shorthand 
that took time to build. Whether you need execution or exploration. 
That gets rebuilt from scratch every session through the first few 
exchanges, and you pay that cost every time without realizing it.

Memory tells Claude who you are. PCSE captures how you two work 
together. Those are different things. Only one of them exists 
right now.

---

## The Problem

Every session starts cold. Memory systems store facts about a user 
— not the interaction pattern itself. The rhythm, the compression 
tolerance, the vocabulary that no longer needs explaining — all of 
it has to be rebuilt from zero.

This has a measurable cost. Early exchanges carry higher overhead 
— more clarification requests, more misreads, more tokens spent 
establishing a baseline before substantive work begins. The 
calibration tax is paid every time.

Existing approaches don't solve it. Conversation state embeddings 
exist in research but aren't portable — they live inside a model 
during inference and disappear when the session ends. User 
embeddings encode the wrong thing — who you are, not how well you 
and another party are communicating. They're profiles, not 
relationship states.

## What PCSE Solves

PCSE extracts the calibration signal from conversation history, 
compresses it into a fixed-length vector, normalizes it for cosine 
comparison, and fingerprints it with a deterministic hash. The 
result is a portable, verifiable artifact encoding the quality of 
a specific two-party interaction pattern — injectable at session 
start to eliminate the cold start overhead tax.

Doesn't matter if you use Claude to write code, make art, plan 
your week, or just think out loud. The working relationship you've 
built shouldn't have to be re-established every time.

---

## Status — v0.1

All four layers shipped. Stdlib-only. No numpy, no torch, no
external dependencies. Python 3.10+ for `str | None` union syntax.

```
Feature space → Scoring rubric → Algorithm → Embedding → Hash
   (defined)      (binary)      (8-dim)     (L2 norm)    (SHA-256)
   v0.1 ✓         v0.1 ✓        v0.1 ✓      v0.1 ✓       v0.1 ✓
```

Every stage runs end-to-end against fixture data committed in `data/`.
The numbers below are reproduced verbatim from those runs.

## Architecture

Four layers, each doing exactly one thing, each handing off to the 
next.

### Layer 1 — Scorer (`src/clarification_overhead.py`)

Reads a session as a list of exchanges and emits a score per 
exchange and an aggregate session score. A clarification request 
is any exchange where either party cannot proceed without 
additional information — binary, either it happened or it didn't.

The scorer detects clarifications via opener phrases (`"could you 
clarify"`, `"what do you mean"`, etc.) and a short-question 
heuristic that is role-aware: short questions on the assistant 
side count as clarifications by default, but on the user side 
they require a back-reference signal (`"you said"`, `"earlier"`, 
`"wait,"`) to avoid false-firing on normal Q&A.

Exchanges may carry an explicit `clarification` boolean to 
override the heuristics — useful for ground-truth testing.

### Layer 2 — Vector (`src/calibration_vector.py`)

Compresses an ordered series of session scores into a fixed-length 
8-dim vector. Same input always produces the same vector — pure 
arithmetic, no randomness, no iteration-order surprises.

| Dim | Name              | Meaning                                       |
|-----|-------------------|-----------------------------------------------|
| 0   | trend_slope       | Least-squares slope of ratio over sessions    |
| 1   | trend_direction   | +1 improving, -1 worsening, 0 flat            |
| 2   | volatility_std    | Population std-dev of ratio across sessions   |
| 3   | volatility_range  | max(ratio) − min(ratio)                       |
| 4   | weighted_recent   | Exponentially-weighted recent score (α=0.7)   |
| 5   | mean_score        | Arithmetic mean of session_score              |
| 6   | latest_score      | session_score of the newest session           |
| 7   | session_count_log | log(1 + n_sessions) — series-length metadata  |

### Layer 3 — Embedding (`src/trust_embedding.py`)

L2-normalizes the comparison view of the raw vector to unit 
length, so cosine similarity reduces to the dot product.

`session_count_log` is excluded from the comparison vector. It 
encodes how much history exists, not what the calibration state 
is. Two pairs of people with identical session counts but opposite 
trajectories were inflating each other's similarity scores. 
Excluding it dropped the cosine separation between an improving 
series and a degrading series from +0.554 to +0.167.

`compare()` returns the raw cosine in [-1.0, 1.0]. Downstream 
consumers can map to [0, 1] via `(cos + 1) / 2` if a non-negative 
score is needed.

### Layer 4 — Hash (`src/trust_hash.py`)

SHA-256 over a canonical JSON serialization of the *full* 8-dim 
vector — `session_count_log` is included here even though it's 
excluded from cosine comparison, because the hash is identity, not 
similarity. Two calibration states that look alike under cosine 
can still be distinct runs, and the hash needs to distinguish them.

Canonical format pins decimal precision (10 places) and dict key 
order (sorted) so identical inputs produce byte-identical canonical 
strings on any machine. Any change to any dim — including metadata 
dims — produces a different hash.

`diff()` accepts two hash records and surfaces which dims changed 
and by how much. The integrity of stored records is verified before 
diffing: if a record's stored hash doesn't match the hash recomputed 
from its stored vector, `diff()` raises.

---

## Validation

Every claim above is exercised by an end-to-end run against 
fixtures in `data/`. The numbers reported are the actual outputs.

### Scorer — `data/sample_exchanges.json`

Five exchanges, two clarifications, three substantive.

```
$ python3 src/clarification_overhead.py
[CLAR] ex-01   assistant:opener:'could you clarify'
[----] ex-02   substantive
[----] ex-03   substantive
[CLAR] ex-04   user:opener:'what do you mean'
[----] ex-05   substantive
ratio                 : 0.4
session score         : 0.6
```

The first version of the scorer flagged three clarifications by 
firing the short-question heuristic on a normal user question. 
That was wrong — the user could proceed; only the assistant was 
ambiguous. The fix was making the short-question rule role-aware: 
require a back-reference signal on the user side.

### Vector — `data/multi_session.json`

Four sessions with monotonically falling overhead.

```
$ python3 src/calibration_vector.py
trend_slope         -0.159520
trend_direction     +1.000000
volatility_std       0.178534
volatility_range     0.472200
weighted_recent      0.725154
mean_score           0.656750
latest_score         0.888900
session_count_log    1.609438
```

Negative slope plus positive direction = ratio is falling, 
calibration is improving.

### Embedding — `data/comparison_test.json`

Two contrasting series: A (improving) and B (degrading).

```
$ python3 src/trust_embedding.py
comparison dim count       : 7
cosine similarity (A vs B) : +0.166618
mapped to [0, 1]           : 0.583309
```

Without the `session_count_log` exclusion this number was +0.554. 
Pulling the metadata dimension out of the comparison vector 
sharpened the separation by ~3.3×.

### Hash — `data/comparison_test.json`

Run the improving series, hash it. Flip one exchange — one 
`clarification` label from false to true in the most recent 
session — and rehash.

```
$ python3 src/trust_hash.py
ORIGINAL  sha-256: 02ac79195f10...
MUTATED   sha-256: 695ff0dea6b6...

hashes match  : False
changed dims  : 6
  trend_slope        -0.2000000000 → -0.1400000000  Δ=+0.0600000000
  volatility_std     +0.2236067977 → +0.1658312395  Δ=-0.0577755582
  volatility_range   +0.6000000000 → +0.4000000000  Δ=-0.2000000000
  weighted_recent    +0.7861034347 → +0.7071456771  Δ=-0.0789577576
  mean_score         +0.7000000000 → +0.6500000000  Δ=-0.0500000000
  latest_score       +1.0000000000 → +0.8000000000  Δ=-0.2000000000
```

One exchange flip rippled through six of eight dimensions. 
`trend_direction` (still +1.0, series still improving overall) 
and `session_count_log` (still 4 sessions) correctly did not 
change. Everything that should have moved did. Everything that 
shouldn't, didn't.

---

## Repo Layout

```
src/
  clarification_overhead.py   Layer 1 — binary scorer
  calibration_vector.py       Layer 2 — 8-dim algorithm
  trust_embedding.py          Layer 3 — L2 normalize + cosine
  trust_hash.py               Layer 4 — SHA-256 fingerprint + diff
data/
  sample_exchanges.json       5-exchange fixture for the scorer
  multi_session.json          4-session fixture for the vector
  comparison_test.json        2-series fixture for embed + hash
scripts/
  build_pdf.py                Renders the PCSE paper as PDF
                              (requires matplotlib, weasyprint)
```

Each script under `src/` is independently runnable and uses its 
own default fixture from `data/`. Pass a path argument to run 
against your own data instead.

## What's Not Built Yet

- **Three more dimensions.** Friction rate, recovery speed, 
  vocabulary convergence. Each gets its own scorer module 
  emitting the same shape the algorithm already consumes.
- **Learned classifiers.** The current scorer is pattern-matching. 
  A small classifier trained on labeled exchanges replaces it 
  without changing anything above the scoring layer.
- **Z-score normalization** per dimension before L2. Right now 
  `trend_slope` is dwarfed by `volatility_range` in the dot 
  product even when slope is more informative.
- **Live conversation parser.** Right now PCSE reads JSON files. 
  Production needs a parser that ingests live exchanges and 
  emits scored sessions on session close.
- **Multi-dyad validation.** The architecture was tuned against 
  one Nick + Claude relationship. Whether the dimensions 
  generalize is an open question that needs more dyads to 
  answer.


## Use with Claude Code

Use Claude Code to generate a PCSE snapshot from your current session or design a calibration schema for a new domain.

```
Read README.md in this repo (pcse). Then:
1. Generate a PCSE snapshot based on our current working relationship in this session —
   capture: communication density, shorthand we've established, execution vs. exploration
   ratio, domain-specific calibration
2. Format it as an injectable session-start artifact I can paste at the top of new conversations
3. Identify which calibration dimensions are most load-bearing for our specific work pattern
```

```
I want to build a PCSE file for a domain I haven't worked in with Claude before.
Read README.md, then help me:
1. Bootstrap a PCSE scaffold for [your domain — e.g., firmware reverse engineering, legal research]
2. Identify what calibration signals I should establish in the first few sessions
3. Write the injection prompt I'll use to load this state at the start of each new session
Domain: [describe here]
```

---
## License

Dual-licensed under either of:

- MIT License ([LICENSE-MIT](LICENSE-MIT))
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))

at your option.

---

*NuClide — Nick + Claude*
