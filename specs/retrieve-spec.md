# Spec: `retrieve()`

**File:** `retriever.py`
**Status:** Spec drafted — review the design decisions below, then implement `retrieve()`

---

## Purpose

Given a user's natural language query, find the most relevant chunks from the vector store using semantic similarity search. Return them ranked by relevance so that `generate_response()` can use them as context.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's natural language question |
| `n_results` | `int` | Maximum number of chunks to return (default: `N_RESULTS` from `config.py`) |

**Output:** `list[dict]`

Each dict in the returned list must contain exactly these keys:

| Key | Type | Description |
|-----|------|-------------|
| `"text"` | `str` | The chunk text |
| `"game"` | `str` | The game name this chunk came from |
| `"distance"` | `float` | Cosine distance score — lower means more similar to the query |

Results should be ordered from most to least relevant (lowest to highest distance). Returns an empty list `[]` if the collection contains no documents.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Query approach

*Describe how you will use `_collection.query()` to find relevant chunks. What arguments will you pass, and why?*

```
Call:

    _collection.query(
        query_texts=[query],
        n_results=n_results,
        include=["documents", "metadatas", "distances"],
    )

- query_texts=[query]: we pass the raw question string (wrapped in a list).
  ChromaDB embeds it on the fly with the SAME all-MiniLM-L6-v2 function used at
  ingestion, so the query vector lands in the same 384-dim space as the stored
  chunks — that shared space is what makes the comparison meaningful.
- n_results: how many nearest chunks to return. Defaults to N_RESULTS (3) from
  config.py so retrieval and the rest of the pipeline stay configured in one
  place.
- include=["documents", "metadatas", "distances"]: we need all three —
  documents = the chunk text to answer from, metadatas = which game each chunk
  came from, distances = the cosine score used to rank and (downstream) judge
  relevance. ids are not requested; we don't need them here.

ChromaDB already returns results sorted ascending by distance, so no manual
sort is required — but I'll still rely on that ordering rather than re-sort.
```

---

### Return structure

*Sketch out what one item in your return list looks like as a concrete example. Where does each field come from in the query results?*

```
One returned item:

    {
        "text": "If you roll a 7, no one collects resources. Any player with "
                "more than 7 cards must discard half ...",
        "game": "Catan",
        "distance": 0.41,
    }

Field sources (results = _collection.query(...)):
    "text"     <- results["documents"][0][i]
    "game"     <- results["metadatas"][0][i]["game"]
    "distance" <- results["distances"][0][i]

Build the list by walking the three parallel inner lists together, e.g. with
zip(results["documents"][0], results["metadatas"][0], results["distances"][0]).
```

---

### Handling the nested result structure

*`_collection.query()` returns nested lists. Describe what index you need to access to get the actual list of results for a single query, and why the nesting exists.*

```
Every field comes back as a list-of-lists. The OUTER list has one entry per
query in query_texts (the API is built to score a batch of queries in one
call); the INNER list holds that one query's ranked results.

We submit a single query, so everything we want is at index [0]:
    results["documents"][0]   # list of chunk texts
    results["metadatas"][0]   # list of metadata dicts
    results["distances"][0]   # list of distances

So the access pattern is results[field][0][i] for the i-th result. Forgetting
the [0] is the classic bug here — you'd be iterating the batch dimension
instead of the results.
```

---

### Relevance threshold

*Will you filter out results above a certain distance score, or return all `n_results` regardless of how relevant they are? What are the tradeoffs of each approach?*

```
Decision: do NOT filter by distance inside retrieve(). Return all n_results,
ranked, with each chunk's distance preserved in the output. The relevance
judgment (what is "good enough" to answer from) is deferred to
generate_response() in Milestone 3.

Why:
- Keeps retrieve() a pure search primitive: "rank and return the n nearest
  chunks." Single responsibility, easy to test and reason about.
- The threshold decision is genuinely the generator's call — that's where
  grounding happens, and per system-design.md a cosine distance above ~0.5
  signals weak relevance. Putting the cutoff there keeps it next to the code
  that acts on it.
- Preserving the distance in the output is what makes the deferral possible:
  the generator can drop weak chunks, or still say "the closest rule I found
  was X, but it doesn't really answer this."

Tradeoff: returning weak matches means the generator MUST be disciplined about
ignoring them — otherwise a high-distance chunk could pull the answer
off-topic. The alternative (filter here, e.g. drop distance > 0.5) guarantees
only strong chunks flow downstream, but risks returning [] and throwing away
the "here's the closest thing I found" option. If filtering is wanted later,
the cleanest extension is an optional max_distance parameter, defaulting to off.
```

---

### Edge cases

*How does your implementation behave when: (a) the collection is empty, (b) the query matches no chunks well, (c) the query matches chunks from multiple games?*

```
(a) Empty collection: guarded up front — if _collection.count() == 0, return []
    immediately (no query call). app.py / generate_response() then shows the
    "couldn't find anything in the loaded rule books" message.

(b) Query matches nothing well: ChromaDB still returns the n_results CLOSEST
    chunks, just with high distances (> ~0.5). We return them as-is with their
    distances rather than inventing a "no match" — recognizing weak relevance
    is the generator's job (see threshold decision), and the distances give it
    the signal it needs.

(c) Query spans multiple games: results can legitimately mix games, since each
    chunk carries its own "game" field and ordering is purely by distance. A
    question two games could answer returns the best chunks regardless of
    source. Caveat: a generic query like "how do you win?" may surface chunks
    from several games, and the top hit might not be the game the user had in
    mind — so the generator must cite the game per chunk and not blur them
    together.
```

---

## Implementation Notes

*Fill this in after implementing, before moving to Milestone 3.*

**Test query and top result returned:**

```
Query: What happens when you run out of disease cubes in Pandemic?
Top result game: Pandemic
Distance score: 0.373
Does it make sense? Yes — all three returned chunks are Pandemic, and the top
one is the loss condition ("...any color of disease cubes runs out when you
need to add one..."), which is exactly the rule the query asks about. The
[0] unwrap and game metadata both came through correctly.

(Cross-checks: "Can two players share a route?" -> Ticket to Ride 0.365, all 3
TtR, top chunk states the double-route rule. "How do you win?" -> Monopoly /
Risk / Ticket to Ride win conditions, ~0.51 — a correct multi-game semantic
match, not a bug.)
```

**One thing about the query results that surprised you:**

```
Distances for *good* matches are higher than expected — clustering around
0.37–0.52, not the 0.1–0.2 the reading describes as "highly relevant." The
cause traces straight back to Milestone 1 chunking: at 300 chars the rules get
split mid-sentence, so even the right chunk only partially overlaps the query
and the embedding similarity drops.

The clearest example is "What happens when you roll a 7?" — the top hit is the
correct Catan chunk (0.466), but it begins mid-sentence ("x, that hex produces
no resources...") and results #2 and #3 are Risk *dice-rolling* chunks (~0.60).
"Roll" and "dice" embed similarly across games, so smaller chunks with less
surrounding context let an unrelated game leak into the top-k. Confirms the
reading's point that weak retrieval is usually a chunking problem, not a query
bug — and is a good argument for revisiting chunk size / adding overlap if M3
answers come out thin.
```
