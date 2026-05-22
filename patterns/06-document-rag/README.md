# 06 — Document RAG

**Retrieve grounded answers from your own documents. Every claim traceable to a source.**

## The problem

Most RAG implementations are a retrieval call followed by a prompt that says "answer using this context." If the context is wrong, the answer is wrong, and nothing tells you why. There's no typed schema for what a retrieval result looks like, no channel governing how requests move through the pipeline, and no session state tracking which knowledge base is active.

## The agentic anti-pattern this replaces

```python
# The common pattern — nothing is typed, nothing is auditable
results = vector_db.search(question, top_k=5)
answer = llm.complete(f"Answer using this context: {results}\n\nQuestion: {question}")
```

## The Agicore approach

- `ENTITY KnowledgeBase` + `ENTITY Document` — typed document store with relationships
- `SESSION retrieval_session` — tracks active knowledge base and query history across interactions
- `PACKET RetrievalRequest` / `PACKET RetrievalResult` — typed contracts for what goes in and comes out of the pipeline
- `CHANNEL retrieval_pipeline` — governs how retrieval requests flow (retry policy, timeout, ordering)
- `ACTION ingest_document` — AI-assisted chunking with keyword extraction
- `ACTION answer_question` — grounded answer that cites sources by document title

## Key declarations

| Declaration | Why it's here |
|-------------|---------------|
| `SESSION` | Maintains active knowledge base context across multiple queries |
| `CHANNEL` | Typed bidirectional message bus — retrieval requests in, grounded results out |
| `PACKET` | Schema for retrieval request and response — if it doesn't match the type, it fails cleanly |
| `ACTION` | Typed AI dispatch with explicit input/output — no ambient state |

## How to compile and run

```bash
cd path/to/agicore/core/compiler
node dist/cli.js generate path/to/agicore-examples/06-document-rag/document_rag.agi --output ~/my-rag-app
cd ~/my-rag-app && npm install && cargo tauri dev
```
