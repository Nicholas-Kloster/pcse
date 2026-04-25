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

## Build Sequence

Feature space → Scoring rubric → Algorithm → Embedding → Hash

## Source Files

- `src/clarification_overhead.py` — scores each exchange binary 
  for clarification events
- `src/calibration_vector.py` — compresses session history into 
  an 8-dim calibration vector
- `src/trust_embedding.py` — L2-normalizes the vector for cosine 
  comparison
- `src/trust_hash.py` — SHA-256 fingerprints the calibration 
  state for portability and drift detection

## Running the Pipeline

python3 src/trust_embedding.py
python3 src/trust_hash.py
python3 src/clarification_overhead.py

---

*NuClide — Nick + Claude*
