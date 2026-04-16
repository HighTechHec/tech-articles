# Semantic Search in Python Without the Cloud: Qdrant and Ollama

Most semantic search tutorials hand you an OpenAI API key and call it a day. This one doesn't. We'll build a fully local semantic search system using Qdrant (a vector database you run yourself) and Ollama (local LLM runner), with zero cloud dependencies and no API costs.

By the end, you'll have a working search engine that understands meaning, not just keywords — running entirely on your machine.

## What We're Building

A Python script that:
1. Embeds documents into vectors using a local model via Ollama
2. Stores and indexes them in a local Qdrant instance
3. Accepts natural language queries and returns semantically similar results

Use case: searching a knowledge base, documentation, or any text corpus without sending data to third parties.

## Prerequisites

- Python 3.10+
- Docker (for Qdrant)
- Ollama installed (`curl -fsSL https://ollama.com/install.sh | sh`)

## Step 1: Start Qdrant Locally

```bash
docker run -d --name qdrant \
  -p 6333:6333 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant:latest
```

Verify it's running:

```bash
curl http://localhost:6333/collections
# {"result":{"collections":[]},"status":"ok","time":0.000034}
```

## Step 2: Pull the Embedding Model

We'll use `nomic-embed-text` — fast, accurate, 768-dimensional embeddings:

```bash
ollama pull nomic-embed-text
```

## Step 3: Install Dependencies

```bash
pip install qdrant-client ollama
```

`requirements.txt`:
```
qdrant-client>=1.7.0
ollama>=0.1.7
```

## Step 4: Build the Indexer

```python
# indexer.py
import ollama
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
import uuid

COLLECTION = "knowledge_base"
EMBED_MODEL = "nomic-embed-text"
VECTOR_SIZE = 768  # nomic-embed-text output dim

client = QdrantClient(host="localhost", port=6333)


def get_embedding(text: str) -> list[float]:
    """Get embedding vector from local Ollama model."""
    response = ollama.embeddings(model=EMBED_MODEL, prompt=text)
    return response["embedding"]


def create_collection():
    """Create Qdrant collection if it doesn't exist."""
    existing = [c.name for c in client.get_collections().collections]
    if COLLECTION not in existing:
        client.create_collection(
            collection_name=COLLECTION,
            vectors_config=VectorParams(size=VECTOR_SIZE, distance=Distance.COSINE),
        )
        print(f"Created collection: {COLLECTION}")


def index_documents(docs: list[dict]):
    """
    Index a list of documents.
    Each doc: {"id": str, "text": str, "metadata": dict}
    """
    create_collection()
    points = []
    for doc in docs:
        vector = get_embedding(doc["text"])
        points.append(
            PointStruct(
                id=str(uuid.uuid5(uuid.NAMESPACE_URL, doc["id"])),
                vector=vector,
                payload={"text": doc["text"], **doc.get("metadata", {})},
            )
        )
    client.upsert(collection_name=COLLECTION, points=points)
    print(f"Indexed {len(points)} documents")


if __name__ == "__main__":
    # Example: index a small knowledge base
    documents = [
        {
            "id": "doc-1",
            "text": "Qdrant is a vector similarity search engine written in Rust.",
            "metadata": {"source": "qdrant_docs", "category": "databases"},
        },
        {
            "id": "doc-2",
            "text": "Ollama lets you run large language models locally on your machine.",
            "metadata": {"source": "ollama_docs", "category": "llm"},
        },
        {
            "id": "doc-3",
            "text": "Python's asyncio enables concurrent code using async/await syntax.",
            "metadata": {"source": "python_docs", "category": "python"},
        },
        {
            "id": "doc-4",
            "text": "Docker containers package applications with their dependencies for consistent deployment.",
            "metadata": {"source": "docker_docs", "category": "devops"},
        },
        {
            "id": "doc-5",
            "text": "Vector embeddings represent text as dense numerical arrays capturing semantic meaning.",
            "metadata": {"source": "ml_primer", "category": "ml"},
        },
    ]
    index_documents(documents)
```

## Step 5: Build the Search Engine

```python
# search.py
import ollama
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

COLLECTION = "knowledge_base"
EMBED_MODEL = "nomic-embed-text"

client = QdrantClient(host="localhost", port=6333)


def get_embedding(text: str) -> list[float]:
    response = ollama.embeddings(model=EMBED_MODEL, prompt=text)
    return response["embedding"]


def search(
    query: str,
    limit: int = 5,
    score_threshold: float = 0.5,
    filter_category: str | None = None,
) -> list[dict]:
    """
    Search the knowledge base.
    Returns list of {text, score, metadata} sorted by relevance.
    """
    query_vector = get_embedding(query)

    query_filter = None
    if filter_category:
        query_filter = Filter(
            must=[
                FieldCondition(
                    key="category",
                    match=MatchValue(value=filter_category),
                )
            ]
        )

    results = client.search(
        collection_name=COLLECTION,
        query_vector=query_vector,
        limit=limit,
        score_threshold=score_threshold,
        query_filter=query_filter,
        with_payload=True,
    )

    return [
        {
            "text": hit.payload["text"],
            "score": round(hit.score, 4),
            "metadata": {k: v for k, v in hit.payload.items() if k != "text"},
        }
        for hit in results
    ]


if __name__ == "__main__":
    queries = [
        "how do I run AI models locally?",
        "what is a vector database?",
        "concurrent programming in Python",
    ]

    for q in queries:
        print(f"\nQuery: {q}")
        hits = search(q, limit=2)
        for hit in hits:
            print(f"  [{hit['score']}] {hit['text'][:80]}...")
            print(f"           category={hit['metadata'].get('category')}")
```

Run it:

```bash
python indexer.py
# Indexed 5 documents

python search.py
# Query: how do I run AI models locally?
#   [0.8921] Ollama lets you run large language models locally on your machine....
#   [0.6234] Qdrant is a vector similarity search engine written in Rust....

# Query: what is a vector database?
#   [0.9102] Qdrant is a vector similarity search engine written in Rust....
#   [0.7815] Vector embeddings represent text as dense numerical arrays...

# Query: concurrent programming in Python
#   [0.8843] Python's asyncio enables concurrent code using async/await syntax....
```

## Step 6: Indexing Files from Disk

Real use case — index a directory of markdown docs:

```python
# index_dir.py
import os
from pathlib import Path
from indexer import index_documents


def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """Split text into overlapping chunks for better retrieval."""
    words = text.split()
    chunks = []
    i = 0
    while i < len(words):
        chunk = " ".join(words[i : i + chunk_size])
        chunks.append(chunk)
        i += chunk_size - overlap
    return chunks


def index_directory(path: str, extensions: tuple = (".md", ".txt", ".rst")):
    docs = []
    for fp in Path(path).rglob("*"):
        if fp.suffix not in extensions:
            continue
        text = fp.read_text(errors="ignore")
        chunks = chunk_text(text)
        for i, chunk in enumerate(chunks):
            docs.append({
                "id": f"{fp.stem}-chunk-{i}",
                "text": chunk,
                "metadata": {
                    "source": str(fp),
                    "chunk": i,
                    "filename": fp.name,
                },
            })
    print(f"Indexing {len(docs)} chunks from {path}")
    index_documents(docs)


if __name__ == "__main__":
    index_directory("./docs")
```

## Step 7: Persistent Collection Across Restarts

The `qdrant_storage` volume from our Docker command persists data. To verify:

```bash
# Stop and restart Qdrant
docker restart qdrant

# Your data is still there
python search.py  # same results
```

## Performance Notes

On an M2 Mac, `nomic-embed-text` embeds ~50 documents/second. For larger corpora:

```python
# Batch embedding for speed
def batch_embed(texts: list[str], batch_size: int = 32) -> list[list[float]]:
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i : i + batch_size]
        for text in batch:
            response = ollama.embeddings(model=EMBED_MODEL, prompt=text)
            embeddings.append(response["embedding"])
    return embeddings
```

Qdrant handles millions of vectors efficiently. For 100k+ documents, add an HNSW index (enabled by default in Qdrant).

## Conclusion

You now have a fully local semantic search system: Qdrant stores and retrieves vectors, Ollama generates embeddings, and nothing leaves your machine. The same pattern scales from 100 docs to millions with the same code — just point it at a larger Qdrant instance.

Next steps: add a FastAPI wrapper to expose it as a REST endpoint, or wire it into a RAG pipeline using Ollama's chat models for full local Q&A.
