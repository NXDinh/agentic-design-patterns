# Knowledge Retrieval (RAG) — Reference

Source: Ch 14, Agentic Design Patterns (Gulli/Sauco).

## TL;DR
Retrieve relevant chunks from an external knowledge base, append to the prompt, then generate. Turns the LLM from closed-book to open-book: fresher, more factual, cite-able, domain-specific.

## When to use
- Answer questions over proprietary / internal docs.
- Need citations or verifiability.
- Data changes more often than you can retrain/fine-tune.
- Want to reduce hallucinations with grounded context.

## Core concepts
- **Embedding** — vector encoding of text; similar meaning ⇒ nearby vectors.
- **Text similarity** — lexical (word overlap) vs. semantic (meaning).
- **Semantic distance** — inverse of similarity; small distance = near-synonym.
- **Chunking** — split docs into retrievable units (section / paragraph / sentence). Too large = noisy; too small = loses context. Respect semantic boundaries (headings, paragraphs).
- **Vector DB** — stores embeddings + metadata; HNSW / IVF for fast ANN search. Options: Pinecone, Weaviate, Chroma, Milvus, Qdrant, pgvector (Postgres), Redis vector, Elasticsearch. FAISS / ScaNN power many of them.

## Retrieval strategies
| Strategy | What | Use when |
|---|---|---|
| Vector (semantic) | ANN over embeddings | Meaning-based match, paraphrases, cross-language. |
| BM25 (keyword) | Term-frequency ranking | Exact terms, codes, names, acronyms. |
| Hybrid | Blend BM25 + vector | Default for most production RAG. |
| GraphRAG | Traverse a knowledge graph | Queries needing multi-hop relationships. |

## The pipeline
1. **Ingest** — load docs from source (files, URLs, DB exports).
2. **Clean + convert** — strip boilerplate; convert PDFs/HTML to text or Markdown.
3. **Chunk** — split with overlap (e.g., 300–800 tokens, 10–15% overlap). Keep headers/metadata on each chunk.
4. **Embed** — run through an embedding model.
5. **Index** — store vectors + metadata + source refs in a vector DB.
6. **Retrieve** — on query, embed it, top-k search, optionally hybrid + rerank.
7. **Augment** — stitch retrieved chunks into the prompt with source tags.
8. **Generate** — LLM answers grounded in that context; include citations.
9. **Reconcile** — schedule re-ingest for changed sources.

## Minimal LangChain pattern
```python
loader = TextLoader(path); docs = loader.load()
chunks = CharacterTextSplitter(chunk_size=500, chunk_overlap=50).split_documents(docs)
vs = Weaviate.from_documents(client=client, documents=chunks, embedding=OpenAIEmbeddings())
retriever = vs.as_retriever()

template = """Answer using the context. If unknown, say so.
Question: {question}
Context: {context}
Answer:"""
prompt = ChatPromptTemplate.from_template(template)
rag_chain = prompt | llm | StrOutputParser()
rag_chain.invoke({"question": q, "context": "\n\n".join(d.page_content for d in retriever.invoke(q))})
```

## Agentic RAG (evolution)
Add a reasoning layer that actively curates retrieval. The agent can:
1. **Validate sources** — prefer the 2025 policy doc over a 2020 blog.
2. **Reconcile conflicts** — €50k proposal vs. €65k final report → pick the authoritative one.
3. **Decompose queries** — "compare our features & pricing to X" → 4 parallel sub-queries, then synthesize.
4. **Detect gaps + use tools** — if internal KB lacks data, call live web search.

Cost: higher complexity, latency, and failure modes (agent can loop or discard useful info).

## GraphRAG
Replace (or augment) the vector store with a knowledge graph. Nodes = entities, edges = relationships. Great for: financial/market relationship queries, scientific KG traversal, multi-hop "how does A connect to C via B." Drawbacks: heavy construction/maintenance cost, less flexible, higher latency.

## Key design decisions
- **Chunk size/overlap** — tune per corpus. Start 500/50 tokens; measure.
- **Top-k** — 3–10 typical. Too many = context dilution.
- **Reranker** — cross-encoder (e.g., Cohere Rerank, BGE reranker) on top-k dramatically improves precision.
- **Distance threshold** — drop chunks below a similarity cutoff to avoid noise.
- **Metadata filters** — filter by date, doc type, permission before vector search.
- **Query rewriting** — expand/decompose the user query before embedding (HyDE, multi-query).
- **Citations** — always surface source IDs with the answer.

## Pitfalls & mitigations
- **Needed info spans multiple chunks** → increase overlap, add hierarchical summaries, or use GraphRAG.
- **Irrelevant chunks retrieved** → add reranker; filter by metadata; use hybrid search.
- **Stale KB** → scheduled re-ingest; watch source timestamps; version chunks.
- **PDF/HTML mess** → normalize to Markdown in ingestion; strip navigation/boilerplate.
- **Contradictory sources** → add agentic validation layer; rank by authority/recency.
- **Latency** → cache embeddings for hot queries; pre-compute summaries per doc.
- **Cost creep** → track tokens-per-answer; compress context (summarize retrieved chunks before injecting).

## Evaluation
- **Retrieval metrics**: recall@k, MRR, nDCG.
- **Generation metrics**: faithfulness (answer grounded in context?), answer relevance, context precision. Tools: RAGAS, TruLens, LangSmith evals.
- Build a small labelled eval set early; re-run on every index change.

## Checklist before shipping RAG
- [ ] Chunking strategy documented and tested on representative docs.
- [ ] Hybrid retrieval (vector + BM25) considered.
- [ ] Reranker evaluated.
- [ ] Metadata filters (date, source, permission).
- [ ] Citations in every answer.
- [ ] Eval set with recall@k and faithfulness scores.
- [ ] Re-ingest schedule / change-data-capture.
- [ ] Permission check: user can only retrieve from docs they're allowed to see.
- [ ] Observability: log query, retrieved IDs, final answer, latency.

## RAG as an MCP server (composition)
A clean production design: put the RAG pipeline behind an MCP server exposing:
- `search_knowledge_base(query, top_k, filters)` — tool.
- `get_document(id)` — tool.
- `knowledge_index_summary` — resource.

Any compliant client (Claude Code, ADK agent, etc.) can now use the same retrieval layer. See `mcp.md`.
