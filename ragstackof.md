# Kubernetes RAG — Stack Overflow + Ollama (`llama3.2:1b`)

A fully local RAG pipeline that searches Stack Overflow for Kubernetes issues and uses **Ollama `llama3.2:1b`** to generate solutions.

---

## Architecture

```
User Input → Stack Overflow Search API → Fetch Top Q&A
           → Embed with Ollama (nomic-embed-text)
           → ChromaDB Vector Store
           → Retrieve Relevant Context
           → Ollama LLM (llama3.2:1b) → Solution
```

---

## Prerequisites

```bash
# 1. Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 2. Pull required models
ollama pull nomic-embed-text     # Embedding model
ollama pull llama3.2:1b          # Generation model (lightweight, fast)

# 3. Install Python dependencies
pip install chromadb requests ollama beautifulsoup4
```

---

## Project Structure

```
k8s-rag/
├── main.py
├── fetcher.py
├── embedder.py
├── retriever.py
└── requirements.txt
```

---

## `requirements.txt`

```txt
chromadb
requests
ollama
beautifulsoup4
```

---

## `fetcher.py` — Fetch Stack Overflow Q&A

```python
import requests
from bs4 import BeautifulSoup

STACK_API = "https://api.stackexchange.com/2.3"


def search_stackoverflow(query: str, top_k: int = 5) -> list[dict]:
    """Search Stack Overflow tagged 'kubernetes' for a given issue."""
    params = {
        "order": "desc",
        "sort": "votes",
        "tagged": "kubernetes",
        "intitle": query,
        "site": "stackoverflow",
        "filter": "withbody",
        "pagesize": top_k,
    }
    resp = requests.get(f"{STACK_API}/search/advanced", params=params)
    resp.raise_for_status()
    items = resp.json().get("items", [])

    results = []
    for item in items:
        qid = item["question_id"]
        body = BeautifulSoup(item.get("body", ""), "html.parser").get_text()
        answers = fetch_answers(qid)
        results.append({
            "id": str(qid),
            "title": item["title"],
            "body": body[:500],
            "answers": answers,
            "link": item["link"],
            "score": item.get("score", 0),
        })
    return results


def fetch_answers(question_id: int, top_k: int = 2) -> list[str]:
    """Fetch top-voted answers for a Stack Overflow question."""
    params = {
        "order": "desc",
        "sort": "votes",
        "site": "stackoverflow",
        "filter": "withbody",
        "pagesize": top_k,
    }
    resp = requests.get(
        f"{STACK_API}/questions/{question_id}/answers", params=params
    )
    resp.raise_for_status()
    return [
        BeautifulSoup(a.get("body", ""), "html.parser").get_text()[:800]
        for a in resp.json().get("items", [])
    ]
```

---

## `embedder.py` — Ollama Embeddings + ChromaDB

```python
import ollama
import chromadb

# In-memory vector store (swap to PersistentClient for persistence)
client = chromadb.Client()
collection = client.get_or_create_collection("k8s_stackoverflow")

EMBED_MODEL = "nomic-embed-text"


def embed_text(text: str) -> list[float]:
    """Generate text embeddings using Ollama."""
    resp = ollama.embeddings(model=EMBED_MODEL, prompt=text)
    return resp["embedding"]


def ingest_documents(docs: list[dict]):
    """Embed Stack Overflow results and store in ChromaDB."""
    ids, embeddings, metadatas, documents = [], [], [], []

    for doc in docs:
        full_text = (
            f"Question: {doc['title']}\n\n"
            f"{doc['body']}\n\n"
            f"Answer:\n" + "\n---\n".join(doc["answers"])
        )
        ids.append(doc["id"])
        embeddings.append(embed_text(full_text))
        metadatas.append({
            "title": doc["title"],
            "link": doc["link"],
            "score": doc["score"],
        })
        documents.append(full_text)

    collection.add(
        ids=ids,
        embeddings=embeddings,
        metadatas=metadatas,
        documents=documents,
    )
    print(f"✅ Ingested {len(docs)} documents into ChromaDB")
```

---

## `retriever.py` — Retrieve Context + Generate Answer

```python
import ollama
from embedder import embed_text, collection

# Using llama3.2:1b — lightweight and fast
LLM_MODEL = "llama3.2:1b"


def retrieve_context(query: str, top_k: int = 3) -> list[dict]:
    """Retrieve the most relevant Stack Overflow Q&A from ChromaDB."""
    query_embedding = embed_text(query)
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k,
        include=["documents", "metadatas", "distances"],
    )
    return [
        {
            "content": doc,
            "title": meta["title"],
            "link": meta["link"],
            "score": meta["score"],
        }
        for doc, meta in zip(
            results["documents"][0], results["metadatas"][0]
        )
    ]


def generate_answer(query: str, context_docs: list[dict]) -> str:
    """Use Ollama llama3.2:1b to generate a Kubernetes solution."""
    context_text = "\n\n---\n\n".join(
        [f"[{d['title']}]\n{d['content']}" for d in context_docs]
    )

    prompt = f"""You are a Kubernetes expert assistant.
Use the Stack Overflow context below to diagnose and solve the user's issue.
Be concise since you are a small model — focus on accuracy.

KUBERNETES ISSUE:
{query}

STACK OVERFLOW CONTEXT:
{context_text}

Provide:
1. Root cause of the issue
2. Step-by-step fix with kubectl commands where applicable
3. The most relevant Stack Overflow link from the context

SOLUTION:"""

    response = ollama.chat(
        model=LLM_MODEL,
        messages=[{"role": "user", "content": prompt}],
    )
    return response["message"]["content"]
```

---

## `main.py` — Entry Point

```python
from fetcher import search_stackoverflow
from embedder import ingest_documents
from retriever import retrieve_context, generate_answer


def run_rag(issue: str):
    print(f"\n🔍 Searching Stack Overflow for: '{issue}'\n")

    # Step 1: Fetch Stack Overflow results
    docs = search_stackoverflow(issue, top_k=5)
    if not docs:
        print("❌ No Stack Overflow results found. Try rephrasing your issue.")
        return

    # Step 2: Embed + store in ChromaDB
    ingest_documents(docs)

    # Step 3: Retrieve top relevant context
    context = retrieve_context(issue, top_k=3)

    # Step 4: Generate answer using llama3.2:1b
    answer = generate_answer(issue, context)

    print("\n" + "=" * 60)
    print("💡 SOLUTION\n")
    print(answer)
    print("\n📚 Stack Overflow Sources:")
    for d in context:
        print(f"  • [{d['title']}]")
        print(f"    {d['link']}")
    print("=" * 60 + "\n")


if __name__ == "__main__":
    issue = input("🐳 Describe your Kubernetes issue: ").strip()
    if issue:
        run_rag(issue)
    else:
        print("⚠️  Please enter a valid issue description.")
```

---

## Run

```bash
# Make sure Ollama is running
ollama serve

# Run the RAG system
python main.py
```

---

## Example Session

```
🐳 Describe your Kubernetes issue: pod stuck in Pending state no nodes available

🔍 Searching Stack Overflow for: 'pod stuck in Pending state no nodes available'

✅ Ingested 5 documents into ChromaDB

============================================================
💡 SOLUTION

1. Root Cause:
   The pod cannot be scheduled because no node satisfies its resource requests
   or node selector/affinity rules.

2. Step-by-step Fix:

   # Check pod events
   kubectl describe pod <pod-name>

   # Check node resource availability
   kubectl describe nodes | grep -A5 "Allocated resources"

   # Check if node has the required label
   kubectl get nodes --show-labels

   # If using nodeSelector, verify it matches a node label
   kubectl label node <node-name> disktype=ssd

   # If resource requests are too high, edit the deployment
   kubectl edit deployment <deployment-name>
   # Lower resources.requests.cpu and resources.requests.memory

3. Reference:
   https://stackoverflow.com/questions/...

📚 Stack Overflow Sources:
  • [Kubernetes pod stuck in Pending - no nodes available]
    https://stackoverflow.com/questions/...
============================================================
```

---

## Model & Component Reference

| Component       | Tool / Model          | Notes                            |
|-----------------|-----------------------|----------------------------------|
| **Embedding**   | `nomic-embed-text`    | Local, no API key needed         |
| **Vector Store**| `chromadb` in-memory  | Zero config, instant setup       |
| **LLM**         | `llama3.2:1b`         | Lightweight ~1.3GB, fast         |
| **Data Source** | Stack Overflow API v2 | Free, no auth for basic search   |
| **Language**    | Python 3.10+          | Type hints used throughout       |

---

## Persist Vector Store (Optional)

To avoid re-embedding on every run, switch to a persistent ChromaDB client:

```python
# In embedder.py — replace:
client = chromadb.Client()

# With:
client = chromadb.PersistentClient(path="./chroma_k8s_db")
```

---

## Tips for Better Results

- **Be specific**: `"CrashLoopBackOff after mounting secret volume"` works better than `"pod crashing"`
- **Include error messages**: Paste the exact Kubernetes error for more accurate retrieval
- **llama3.2:1b limitations**: It's a small model — it excels at focused, factual answers but may miss complex multi-step reasoning. For harder issues, switch to `llama3.2:3b` or `llama3`
- **Rate limits**: Stack Overflow API allows ~300 unauthenticated requests/day. Add `&key=YOUR_KEY` to params for higher limits
