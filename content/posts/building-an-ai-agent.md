+++
title = "Building an AI Agent: RAG, Hybrid Search, Semantic Cache, and Memory"
date = "2026-07-07"
draft = false
tags = ["python", "rag", "llms", "redis", "fastapi"]
categories = ["engineering"]
description = "A small documentation assistant built with FastAPI, Redis vector search, OpenAI embeddings, semantic answer caching, session rewriting, long-term memory, and cost metrics."
[cover]
  hidden = true
+++

Most RAG demos stop at the happy path: split a PDF, embed the chunks, retrieve the nearest neighbors, put them in a prompt, call a model. That is the right starting point, but it leaves out the parts that make the system feel like an actual assistant instead of a stateless search box.

I built a small agent in Python to explore those missing pieces. It is a FastAPI service with two main endpoints: `/ingest` for uploading documentation and `/ask` for asking questions against it. Under the hood it uses Redis Stack via `redisvl` for vector and full-text search, OpenAI for embeddings and answer generation, and Redis again for sessions, semantic cache entries, long-term memory, and metrics.

The interesting part is not that it answers questions from documents. The interesting part is what has to happen around that loop: deduplicating chunks, combining vector and keyword retrieval, rewriting follow-up questions before cache lookup, deciding when a cached answer is "close enough", preserving conversational context without poisoning retrieval, and measuring whether the cache actually saves latency and money.

---

## The shape of the app

The public API is deliberately small:

```python
@app.post("/ingest")
async def ingest(file: UploadFile = File(...)):
    text = await load_document(file)
    chunks = split_text(text)
    embeddings = await create_embeddings([c["text"] for c in chunks])
    stats = await store_in_redis(chunks, embeddings, source=file.filename or "unknown")
    return {"filename": file.filename, "chars": len(text), **stats}

@app.post("/ask")
async def ask(req: AskRequest):
    ...
```

`/ingest` turns a document into searchable chunks. `/ask` does everything else: session lookup, optional question enrichment, cache lookup, retrieval, answer generation, memory extraction, cache insertion, and metrics.

That sounds like a lot for one endpoint, but the sequence is useful because each step depends on the previous one:

1. Load recent session messages.
2. Store the user's raw question in the session.
3. If this is an existing session, rewrite the question into a standalone form.
4. Embed that rewritten question.
5. Try a semantic cache lookup.
6. If the cache misses, retrieve document chunks with vector search plus full-text search.
7. Generate an answer using only the retrieved context.
8. Store the answer in the session and semantic cache.
9. Record latency, cache hit rate, and estimated cost.

The rewritten question is the key design choice in the middle. A follow-up like "what about the retry behavior?" is a bad cache key and a bad retrieval query unless you know what "the" refers to. The app stores the original user text for conversation history, but it embeds the enriched standalone question for retrieval and cache lookup.

## Ingestion: boring on purpose

The ingestion path supports PDFs and plain text. PDFs go through PyMuPDF page by page:

```python
if file.content_type == "application/pdf" or (file.filename or "").lower().endswith(".pdf"):
    with pymupdf.open(stream=contents, filetype="pdf") as doc:
        pages = []
        for i, page in enumerate(doc):
            pages.append((i, page.get_text()))
        return pages
```

Plain text gets decoded as UTF-8. From there, both formats are normalized into a list of `(page_number, page_text)` pairs and passed to the same splitter.

The splitter is intentionally simple: paragraph-aware chunks around 2000 characters, with a 200-character overlap. There are fancier strategies: token-aware splitting, markdown-aware splitting, section-title propagation, table extraction, code-block preservation. I skipped all of that at first because the point of the project was to make the retrieval and caching path visible. A boring splitter is easier to reason about while the rest of the system is still moving.

Each chunk gets a deterministic ID from a SHA-256 hash of its stripped text:

```python
def make_chunk_id(text: str) -> str:
    normalized = text.strip().encode("utf-8")
    digest = hashlib.sha256(normalized).hexdigest()
    return f"chunk:{digest}"
```

That one line changes the behavior of re-ingestion. Uploading the same document twice does not produce duplicate vectors. The API can report `received_chunks`, `inserted_chunks`, and `skipped_duplicates`, which is a small operational detail that matters once you start testing with the same files repeatedly.

The chunk is stored with:

- the raw text
- the source filename
- the page number
- the chunk index
- a `text-embedding-3-small` embedding serialized as `float32` bytes

Redis holds both the text index and the vector index. The vector field is configured with 1536 dimensions, cosine distance, `float32`, and the `flat` algorithm. For a small local project, flat search is the simplest thing that works. For a larger corpus, the index choice becomes a real performance decision.

## Retrieval: vector search is not enough

The retrieval path does a vector query first:

```python
query = VectorQuery(
    vector=embedding,
    vector_field_name="embedding",
    return_fields=["text", "source", "page", "chunk_index"],
    num_results=3,
    filter_expression=filter_expression,
)
```

Then it filters results by distance:

```python
results = [r for r in results if float(r["vector_distance"]) < VECTOR_DISTANCE_THRESHOLD]
```

That threshold is doing important work. Without it, the app will always retrieve something, even when the corpus does not contain an answer. The model then receives irrelevant context and is asked to answer "using only the information in the provided context," but the context itself is already misleading. A distance threshold gives the system permission to retrieve nothing.

After vector search, the app also runs a full-text query:

```python
query = TextQuery(
    text=question,
    text_field_name="text",
    return_fields=["text", "source", "page", "chunk_index"],
    num_results=3,
    filter_expression=filter_expression,
    stopwords=None,
)
```

The two result sets are merged by chunk identity. A chunk that appears in both is marked as `hybrid`; vector-only matches are marked as `vector`; keyword-only matches are marked as `full_text`.

This is the first place where the RAG demo version breaks down in practice. Vector search is good at semantic similarity, but it can miss exact things: function names, CLI flags, error codes, class names, config keys. Full-text search is crude, but it is very good at finding the literal symbol the user typed. Hybrid retrieval is not glamorous; it just fixes a real failure mode.

The `/ask` response includes source metadata:

```python
"sources": [
    {
        "source": c.get("source"),
        "page": c.get("page"),
        "chunk_index": c.get("chunk_index"),
        "match_type": c.get("match_type"),
        "vector_distance": c.get("vector_distance"),
        "text_score": c.get("text_score"),
    }
    for c in chunks
]
```

That metadata is not only for display. It is also how you debug retrieval. When an answer is bad, the first question is not "why did the model say that?" It is "what context did we give it?"

## The answer prompt

The answer generation prompt is strict:

```python
SYSTEM_PROMPT = """
    You are a documentation assistant.
    This is the long-term memory context: {additional_context}.
    Answer the user's question using ONLY the information in the provided context.
    For each piece of information in your answer, cite the source file it came from using the format (source: <filename>).
    If the context does not contain enough information to answer the question, say so honestly -- do not make anything up.
    Keep answers to 1-3 sentences unless the question requires more detail.
"""
```

The important phrase is "using ONLY the information in the provided context." This does not magically prevent hallucination, but it gives the model a narrow job. The user message then includes the retrieved chunks as a `Context:` block followed by the question.

There is a subtle tension here: the app also injects long-term memory into the system prompt. That memory is useful for preferences and stable user context, but it should not become a second source of factual document content. The memory extractor explicitly avoids storing facts about external documents unless they describe the user's own project or work. That boundary matters. Otherwise, yesterday's answer can silently become today's source.

The model call records token usage and estimates cost:

```python
estimated_cost = (
    prompt_tokens * INPUT_PRICE_PER_TOKEN
    + completion_tokens * OUTPUT_PRICE_PER_TOKEN
)
redis_client.incrbyfloat(M_COST_SPENT, estimated_cost)
```

This is approximate, but it turns "LLM calls cost money" into a number the app can observe.

## Semantic cache: caching answers, not strings

A normal cache key would be the exact question text. That is almost useless for a natural-language interface. These should be cache hits:

- "How do I configure Redis?"
- "How is Redis configured?"
- "Where does the Redis URL come from?"

They will not be exact string matches, but their embeddings should be close if they are asking the same thing. So the app stores cache entries as `{question, response, embedding}` in a second Redis vector index:

```python
cache_index.load(
    [{
        "question": question,
        "response": response,
        "embedding": np.array(embedding, dtype=np.float32).tobytes(),
    }],
)
```

Lookup is another vector query, with a much tighter threshold than document retrieval:

```python
if distance < CACHE_DISTANCE_THRESHOLD:
    return results[0]
```

That threshold is the entire product decision. Too low, and the cache rarely hits. Too high, and the app returns a confident answer to the wrong question. Document retrieval can tolerate a loose threshold because the model still has a chance to say "the context does not contain enough information." A semantic answer cache has no such safety valve. On a cache hit, the stored answer goes straight back to the user.

This is why the question enrichment step runs before cache lookup. If the session contains:

```text
User: How does ingestion work?
Assistant: ...
User: What about duplicates?
```

the cache should not store or search for "What about duplicates?" It should search for something like "How does the ingestion flow handle duplicate document chunks?" The enriched question makes cache entries more reusable and less ambiguous.

There is still a hard limitation: cached answers can go stale when documents are re-ingested or when the source corpus changes. This project has a `/cache/flush` endpoint, which is fine for a local tool, but a production version would need cache invalidation tied to document versions or source hashes.

## Sessions: short-term memory with an expiration date

Session state is just a Redis list:

```python
def generate_session_key(session_id: str) -> str:
    return f"session:{session_id}:messages"

def push_message(key: str, role: str, content: str) -> None:
    redis_client.rpush(key, json.dumps({"role": role, "content": content}))
    redis_client.expire(key, SESSION_TTL_SECONDS)
```

The app reads the last ten messages and gives them to both the question rewriter and the answer model. It also sets a 24-hour TTL. That is a sensible default for a documentation assistant: long enough for a work session, short enough that abandoned conversations do not accumulate forever.

The distinction between session memory and long-term memory is important:

- Session memory is the recent conversation transcript.
- Long-term memory is durable user preference or project context extracted from questions.

They are stored differently and used differently. Session memory helps resolve "it", "that", and "the same." Long-term memory helps answer in a way that respects stable preferences. Mixing those two concepts usually leads to messy behavior: either every passing detail becomes permanent, or useful preferences disappear when the session expires.

## Long-term memory: useful, but easy to over-store

The memory extractor asks a small model to decide whether a user message contains durable context:

```python
if there is new durable memory to store:
{
  "should_store": true,
  "preference": "one concise memory sentence"
}
```

The prompt is more interesting than the code. It says to store preferences, recurring work habits, technologies, project-level context, stable constraints, dislikes, and long-lived goals. It says not to store generic questions, one-off tasks, temporary facts, low-confidence guesses, duplicate memories, or arbitrary facts about external documents.

That is the right instinct. Long-term memory is not a transcript. It is a small set of durable facts that should improve future interactions. If the memory store becomes a dumping ground, it starts polluting prompts and retrieval with stale assumptions.

The current implementation stores memories in a Redis list and injects all of them into the system prompt. That is fine while the list is short. The next step would be a memory index with retrieval, deletion, and maybe confidence or provenance. At some point, memory needs the same care as documents: search, deduplication, source tracking, and expiration rules.

## Metrics: the feedback loop

The app tracks:

- total requests
- cache hits and misses
- average cached latency
- average uncached latency
- estimated cost spent
- estimated cost saved

The cost-saved calculation is simple: on a cache hit, it estimates savings as average cost per previous miss. That is not perfect, but it is good enough to tell whether the cache is doing anything.

The more important point is that the system measures cache behavior at all. Semantic caching introduces risk. You should get something back for accepting that risk: lower latency, lower cost, or both. If the hit rate is low, the cache is complexity without much payoff. If the hit rate is high but users report wrong answers, the threshold is probably too loose. Metrics are what let you tune that tradeoff instead of guessing.

## The rough edges

This project is small enough that some rough edges are visible, which is useful.

The Redis indexes are created on import:

```python
redis_index.create(overwrite=True)
cache_index.create(overwrite=True)
```

That is convenient during development and dangerous if you expect persistence. A real deployment should make index creation an explicit migration or startup step that does not casually overwrite existing indexes.

The chunker tracks page numbers, but when a chunk crosses a page boundary the page metadata can become fuzzy. That is tolerable for basic citations, but it is not good enough for precise page-level attribution.

The cache has a manual flush endpoint, but no automatic invalidation when the document corpus changes. If you ingest a corrected manual, an old cached answer can still survive. The chunk hash solves duplicate ingestion; it does not solve answer freshness.

The answer generator calls `add_to_memory(question)` before generating the answer. That means even questions that cannot be answered from the corpus can still influence memory. The memory prompt is conservative, but this is a place to be careful. Durable memory should probably be tied to user intent, not merely the fact that a question passed through the system.

Finally, the system prompt asks for citations by source file, but the generated answer only sees the source filename in the context header, not page or chunk metadata. The API returns source metadata separately, so the information exists; the prompt could include richer chunk labels if citation precision mattered.

## What this project teaches

The core RAG loop is short. The surrounding system is where the engineering lives.

Ingestion needs stable IDs so repeated uploads do not duplicate data. Retrieval needs both semantic and lexical search because users ask with concepts and exact symbols. Cache lookup needs rewritten questions because conversation is full of references. Memory needs a strict storage policy because not every sentence deserves to become durable context. Metrics need to exist because otherwise you cannot tell whether your cache is a win or a liability.

The biggest lesson is that a documentation assistant is less like a single model call and more like a small information system. The model writes the final sentence, but the quality of that sentence is mostly determined before generation starts: what got chunked, what got retrieved, what got cached, what got remembered, and what the system chose to measure.

That is why building one of these by hand is worth it. You stop thinking of RAG as "put documents in the prompt" and start seeing the actual boundaries: retrieval quality, answer freshness, cache correctness, memory hygiene, and operational feedback. Those are the parts that decide whether the assistant is useful after the demo.
