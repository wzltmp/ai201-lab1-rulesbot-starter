# RulesBot — Planning Doc

Use this file to record your design decisions as you work through the lab.
There are no wrong answers — write enough that you could explain your reasoning to another group.

---

## Chunking Strategy

**Chunk size:** 300 characters (character-based sliding window).

**Overlap:** 50 characters between adjacent chunks. The window advances by
`chunk_size - overlap` = 250 characters per step, so each chunk shares its
last 50 characters with the start of the next. Chunks shorter than 50
characters (after stripping) are discarded as noise.

**Why this strategy fits rule book text:**
Rule book text is semantically dense — a single rule is usually only 1–3
sentences, which sits comfortably inside a 300-character window. That makes
~300 characters roughly "one rule," which is the right unit of retrieval for
targeted questions like "What happens when you roll a 7?" Smaller chunks would
fragment a single rule across multiple entries; larger ones would merge
unrelated rules into one chunk and blur retrieval. The 50-character overlap
guards the boundary case where a rule straddles two chunks, so it can still be
retrieved intact, while the 50-character minimum drops whitespace artifacts and
bare section headers that carry no semantic signal. The main tradeoff: this
splitter ignores sentence/paragraph boundaries, so a chunk can begin
mid-sentence or split a long rule or numbered list. A sentence- or
paragraph-aware splitter would handle those cases better at the cost of more
complexity.

---

## Retrieval Observations

After implementing retrieval, try these test queries and record what comes back:

| Query | Top result game | Does it make sense? |
|-------|----------------|---------------------|
| "How do you win?" | Monopoly (0.507) | Yes — top-3 are Monopoly / Risk / Ticket to Ride win conditions. "Win" is semantically close to every game's victory rule, so a spread across games is correct behavior, not a bug. |
| "What happens when you roll a 7?" | Catan (0.466) | Mostly — right game on top (Catan robber/resource rule), but the chunk starts mid-sentence and results #2–#3 are Risk dice-rolling (~0.60). Distance is only moderate. |
| "Can two players share a route?" | Ticket to Ride (0.365) | Yes — all 3 results are Ticket to Ride; the top chunk directly states the double-route rule. Cleanest of the three queries. |

**Anything surprising?**

Good matches land around 0.37–0.52, not the 0.1–0.2 "highly relevant" range the
reading describes. The reason is the Milestone 1 chunking: 300-char windows split
rules mid-sentence, which lowers similarity even for the correct chunk and lets a
loosely-related game (Risk dice rolling for "roll a 7") leak into the top-k. So
retrieval quality here is gated by chunk size, not the query code — exactly the
diagnosis the lab warns about.


---

## Response Quality

After implementing generation, try 2–3 questions and assess the answers:

| Query | Answer accurate? | Properly grounded? | Cited the right game? |
|-------|-----------------|-------------------|----------------------|
| | | | |
| | | | |

**What would you change about the prompt to improve grounding?**

