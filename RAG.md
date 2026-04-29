# Workshop: Build a RAG System with Ollama
### Load Documents · Embed & Index · Query with Context

No cloud required. Everything runs locally from the terminal or browser.

---

## Prerequisites Checklist

| Tool | Version | Check Command |
|------|---------|---------------|
| Python | >= 3.11 | `python --version` |
| Ollama | Latest | `ollama --version` |
| pip | Latest | `pip --version` |
| Git | >= 2.x | `git --version` |

---

## What We're Building

```
+---------------------+       +---------------------+
|   cli.py (terminal) |       |  app.py (browser UI) |
|   or                |       |  http://localhost:5000|
+----------+----------+       +---------+-----------+
           |                            |
           +------------+---------------+
                        |
           +------------v------------+
           |       tools.py          |
           |  - load_documents       |
           |  - embed_and_index      |
           |  - retrieve_chunks      |
           |  - answer_with_context  |
           +------------+------------+
                        |
           +------------v------------+
           |    Ollama (llama3.2:1b) |
           |  + nomic-embed-text     |
           |    localhost:11434      |
           +-------------------------+
```

Two ways to run:
- **Option A** — `cli.py` : interactive terminal menu
- **Option B** — `app.py` : browser UI using Flask

---

## How RAG Works (Concept)

```
  Your Documents           User Question
       |                        |
       v                        v
 [Chunk & Embed]         [Embed Question]
       |                        |
       v                        v
  [Vector Index]  ------>  [Similarity Search]
                                |
                                v
                     [Top-K Relevant Chunks]
                                |
                                v
                     [Ollama: Question + Chunks]
                                |
                                v
                        [Grounded Answer]
```

RAG = **Retrieval** (find relevant chunks) + **Augmented** (add them to the prompt) + **Generation** (LLM answers using that context).

---

## Project Structure

```
rag-tool/
├── tools.py           <- Core logic (embedding + retrieval + generation)
├── cli.py             <- Terminal interface (Option A)
├── app.py             <- Browser UI with Flask (Option B)
├── requirements.txt   <- Dependencies
├── docs/              <- Put your .txt or .md documents here
└── README.md
```

---

## Part 1 — Setup and Installation

### Step 1.1 — Create the Project

```bash
mkdir rag-tool
cd rag-tool
python3 -m venv venv

# Activate
# macOS/Linux:
source venv/bin/activate

# Create docs folder for your documents
mkdir docs
```

### Step 1.2 — Install Dependencies

Create `requirements.txt`:

```txt
httpx>=0.27.0
flask>=3.0.0
numpy>=1.26.0
```

Install:

```bash
pip install -r requirements.txt
```

### Step 1.3 — Pull the Ollama Models

We need two models: one for **embedding** documents and one for **generating** answers.

```bash
# Start Ollama
ollama serve &

# Pull the embedding model (approx 274MB)
ollama pull nomic-embed-text

# Pull the generation model (approx 1.3GB)
ollama pull llama3.2:1b

# Verify both are available
ollama list
```

### Step 1.4 — Add Sample Documents

Add a few plain-text files to the `docs/` folder to test with. Example:

```bash
cat > docs/company_policy.txt << 'EOF'
Return Policy
Customers may return any product within 60 days of purchase with a receipt.
Items must be in original condition. Electronics have a 30-day return window.
Refunds are processed within 5-7 business days to the original payment method.

Shipping Policy
Standard shipping takes 3-5 business days. Express shipping takes 1-2 days.
Free standard shipping is available on orders over $50. We ship to all 50 states.

Support Hours
Our customer support team is available Monday to Friday, 9am to 6pm EST.
You can reach us at support@example.com or call 1-800-555-0100.
EOF

cat > docs/product_faq.txt << 'EOF'
Q: What payment methods do you accept?
A: We accept Visa, Mastercard, American Express, PayPal, and Apple Pay.

Q: Can I change my order after placing it?
A: Orders can be modified within 1 hour of placement. Contact support immediately.

Q: Do you offer international shipping?
A: Currently we only ship within the United States. International shipping is coming soon.

Q: How do I track my order?
A: You will receive a tracking email within 24 hours of your order shipping.
EOF
```

---

## Part 2 — Build the Core Tools

All shared logic lives in `tools.py`. Both `cli.py` and `app.py` import from it.

Create `tools.py`:

```python
# tools.py
import os
import json
import math
import httpx
from pathlib import Path

OLLAMA_BASE_URL  = "http://localhost:11434"
EMBED_MODEL      = "nomic-embed-text"
DEFAULT_MODEL    = "llama3.2:1b"
CHUNK_SIZE       = 300        # words per chunk
CHUNK_OVERLAP    = 50         # words of overlap between chunks
TOP_K            = 3          # number of chunks to retrieve
INDEX_FILE       = "index.json"


# --------------------------------------------------
# Document loading
# --------------------------------------------------

def load_documents(docs_dir: str) -> list[dict]:
    """
    Load all .txt and .md files from a directory.
    Returns a list of {filename, content} dicts.
    """
    path = Path(docs_dir)
    if not path.exists():
        return []

    docs = []
    for ext in ("*.txt", "*.md"):
        for f in sorted(path.glob(ext)):
            try:
                content = f.read_text(encoding="utf-8").strip()
                if content:
                    docs.append({"filename": f.name, "content": content})
            except Exception as e:
                print(f"[WARN] Could not read {f.name}: {e}")
    return docs


def chunk_text(text: str, chunk_size: int = CHUNK_SIZE, overlap: int = CHUNK_OVERLAP) -> list[str]:
    """
    Split text into overlapping word-based chunks.
    """
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = start + chunk_size
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        if end >= len(words):
            break
        start += chunk_size - overlap
    return chunks


# --------------------------------------------------
# Embedding
# --------------------------------------------------

def embed_text(text: str, model: str = EMBED_MODEL) -> list[float]:
    """
    Call Ollama's embedding endpoint for a single string.
    Returns a vector (list of floats).
    """
    try:
        with httpx.Client(timeout=60.0) as client:
            r = client.post(
                f"{OLLAMA_BASE_URL}/api/embeddings",
                json={"model": model, "prompt": text}
            )
            r.raise_for_status()
            return r.json().get("embedding", [])
    except httpx.ConnectError:
        print("[ERROR] Ollama is not running. Run: ollama serve")
        return []
    except Exception as e:
        print(f"[ERROR] Embedding failed: {e}")
        return []


def cosine_similarity(a: list[float], b: list[float]) -> float:
    """
    Compute cosine similarity between two vectors.
    """
    if not a or not b:
        return 0.0
    dot   = sum(x * y for x, y in zip(a, b))
    mag_a = math.sqrt(sum(x * x for x in a))
    mag_b = math.sqrt(sum(x * x for x in b))
    if mag_a == 0 or mag_b == 0:
        return 0.0
    return dot / (mag_a * mag_b)


# --------------------------------------------------
# Index: build and persist
# --------------------------------------------------

def embed_and_index(docs_dir: str, index_path: str = INDEX_FILE) -> dict:
    """
    Load documents, chunk them, embed each chunk, and save the index.
    Returns a summary of what was indexed.
    """
    docs = load_documents(docs_dir)
    if not docs:
        return {"error": f"No .txt or .md files found in: {docs_dir}"}

    index   = []   # list of {text, filename, embedding}
    total_chunks = 0

    for doc in docs:
        chunks = chunk_text(doc["content"])
        for chunk in chunks:
            vec = embed_text(chunk)
            if vec:
                index.append({
                    "text":      chunk,
                    "filename":  doc["filename"],
                    "embedding": vec,
                })
                total_chunks += 1

    # Persist to disk (embeddings are expensive — don't recompute every run)
    Path(index_path).write_text(json.dumps(index), encoding="utf-8")

    return {
        "documents_loaded": len(docs),
        "filenames":        [d["filename"] for d in docs],
        "chunks_indexed":   total_chunks,
        "index_saved_to":   index_path,
    }


def load_index(index_path: str = INDEX_FILE) -> list[dict]:
    """
    Load a previously built index from disk.
    """
    p = Path(index_path)
    if not p.exists():
        return []
    try:
        return json.loads(p.read_text(encoding="utf-8"))
    except Exception:
        return []


# --------------------------------------------------
# Retrieval
# --------------------------------------------------

def retrieve_chunks(query: str, index_path: str = INDEX_FILE, top_k: int = TOP_K) -> list[dict]:
    """
    Embed the query, then return the top-K most similar chunks.
    Each result: {text, filename, score}
    """
    index = load_index(index_path)
    if not index:
        return []

    query_vec = embed_text(query)
    if not query_vec:
        return []

    scored = []
    for entry in index:
        score = cosine_similarity(query_vec, entry["embedding"])
        scored.append({
            "text":     entry["text"],
            "filename": entry["filename"],
            "score":    round(score, 4),
        })

    scored.sort(key=lambda x: x["score"], reverse=True)
    return scored[:top_k]


# --------------------------------------------------
# Generation with context (RAG)
# --------------------------------------------------

def answer_with_context(query: str, index_path: str = INDEX_FILE, model: str = DEFAULT_MODEL) -> dict:
    """
    Full RAG pipeline:
      1. Retrieve relevant chunks
      2. Build a prompt with those chunks as context
      3. Ask Ollama to answer using only that context
    Returns {answer, sources, chunks_used}
    """
    chunks = retrieve_chunks(query, index_path)
    if not chunks:
        return {
            "answer":      "No indexed documents found. Run 'Embed & Index' first.",
            "sources":     [],
            "chunks_used": 0,
        }

    context = "\n\n---\n\n".join(
        f"[Source: {c['filename']}]\n{c['text']}" for c in chunks
    )

    system = (
        "You are a helpful assistant. "
        "Answer the user's question using ONLY the context provided below. "
        "If the answer is not in the context, say 'I don't have that information.' "
        "Be concise and accurate."
    )

    prompt = (
        f"Context:\n{context}\n\n"
        f"Question: {query}\n\n"
        "Answer:"
    )

    answer  = ollama_chat(prompt, system=system, model=model)
    sources = list({c["filename"] for c in chunks})

    return {
        "answer":      answer,
        "sources":     sources,
        "chunks_used": len(chunks),
    }


# --------------------------------------------------
# Ollama helpers
# --------------------------------------------------

def ollama_chat(prompt: str, system: str = "", model: str = DEFAULT_MODEL) -> str:
    payload = {"model": model, "prompt": prompt, "stream": False}
    if system:
        payload["system"] = system
    try:
        with httpx.Client(timeout=120.0) as client:
            r = client.post(f"{OLLAMA_BASE_URL}/api/generate", json=payload)
            r.raise_for_status()
            return r.json().get("response", "").strip()
    except httpx.ConnectError:
        return "[ERROR] Ollama is not running. Run: ollama serve"
    except Exception as e:
        return f"[ERROR] {e}"


def list_models() -> list:
    try:
        with httpx.Client(timeout=10.0) as client:
            r = client.get(f"{OLLAMA_BASE_URL}/api/tags")
            r.raise_for_status()
            return [m["name"] for m in r.json().get("models", [])]
    except Exception:
        return []


def index_exists(index_path: str = INDEX_FILE) -> bool:
    return Path(index_path).exists()


def index_stats(index_path: str = INDEX_FILE) -> dict:
    index = load_index(index_path)
    if not index:
        return {"indexed": False}
    files = list({e["filename"] for e in index})
    return {
        "indexed":        True,
        "total_chunks":   len(index),
        "files_indexed":  files,
        "index_file":     index_path,
    }
```

---

## Part 3 — Option A: Terminal CLI

Create `cli.py`:

```python
# cli.py
import sys
import tools

DIVIDER  = "-" * 60
DOCS_DIR = "docs"

def prompt(msg: str) -> str:
    return input(f"\n{msg}: ").strip()

def show(title: str, content):
    print(f"\n{DIVIDER}")
    print(f"  {title}")
    print(DIVIDER)
    if isinstance(content, dict):
        for k, v in content.items():
            if isinstance(v, list):
                print(f"  {k}:")
                for item in v:
                    print(f"    - {item}")
            else:
                print(f"  {k}: {v}")
    else:
        print(content)

def menu():
    print("\n" + DIVIDER)
    print("  RAG Tool — powered by Ollama llama3.2:1b")
    print(DIVIDER)
    print("  1. Load & preview documents")
    print("  2. Embed & index documents")
    print("  3. Show index stats")
    print("  4. Retrieve chunks for a query")
    print("  5. Ask a question (full RAG)")
    print("  6. List Ollama models")
    print("  0. Exit")
    return input("\n  Choose: ").strip()

def main():
    while True:
        choice = menu()

        if choice == "0":
            print("Goodbye.")
            sys.exit(0)

        elif choice == "1":
            docs = tools.load_documents(DOCS_DIR)
            if not docs:
                print(f"\n  No documents found in '{DOCS_DIR}/'")
                print(f"  Add .txt or .md files there and try again.")
            else:
                show("Documents Found", {
                    "count": len(docs),
                    "files": [d["filename"] for d in docs],
                })
                preview = docs[0]["content"][:400]
                print(f"\n  Preview of '{docs[0]['filename']}':\n{preview}...")

        elif choice == "2":
            docs_dir = prompt(f"Docs folder (default: {DOCS_DIR})") or DOCS_DIR
            print("\n  Embedding and indexing... this may take a moment.")
            result = tools.embed_and_index(docs_dir)
            if "error" in result:
                print(f"\n  Error: {result['error']}")
            else:
                show("Index Built", result)

        elif choice == "3":
            stats = tools.index_stats()
            if not stats["indexed"]:
                print("\n  No index found. Run option 2 first.")
            else:
                show("Index Stats", stats)

        elif choice == "4":
            if not tools.index_exists():
                print("\n  No index found. Run option 2 first.")
                continue
            query = prompt("Query")
            chunks = tools.retrieve_chunks(query)
            if not chunks:
                print("\n  No results found.")
            else:
                print(f"\n  Top {len(chunks)} chunks retrieved:\n")
                for i, c in enumerate(chunks, 1):
                    print(f"  [{i}] Score: {c['score']}  File: {c['filename']}")
                    print(f"      {c['text'][:200]}...\n")

        elif choice == "5":
            if not tools.index_exists():
                print("\n  No index found. Run option 2 first.")
                continue
            query = prompt("Your question")
            print("\n  Querying Ollama with retrieved context...")
            result = tools.answer_with_context(query)
            show("RAG Answer", {
                "Answer":      result["answer"],
                "Sources":     result["sources"],
                "Chunks used": result["chunks_used"],
            })

        elif choice == "6":
            models = tools.list_models()
            show("Available Ollama Models",
                 "\n  ".join(models) if models else "No models found")

        else:
            print("  Invalid choice. Try again.")

if __name__ == "__main__":
    main()
```

Run it:

```bash
python cli.py
```

---

## Part 4 — Option B: Browser UI

Create `app.py`:

```python
# app.py
from flask import Flask, request, jsonify, render_template_string
import tools

app   = Flask(__name__)
DOCS_DIR = "docs"

HTML = """
<!DOCTYPE html>
<html>
<head>
  <title>RAG Tool</title>
  <style>
    body { font-family: monospace; max-width: 900px; margin: 40px auto; padding: 0 20px; background: #f5f5f5; }
    h1   { font-size: 1.4em; border-bottom: 2px solid #333; padding-bottom: 8px; }
    .row { display: flex; gap: 10px; margin-bottom: 12px; }
    input[type=text] { flex: 1; padding: 8px; font-family: monospace; font-size: 0.95em; }
    button { padding: 8px 18px; background: #333; color: #fff; border: none; cursor: pointer; font-family: monospace; }
    button:hover { background: #555; }
    button.primary { background: #1a56db; }
    button.primary:hover { background: #1e429f; }
    pre  { background: #fff; border: 1px solid #ccc; padding: 16px; white-space: pre-wrap; word-break: break-word;
           max-height: 500px; overflow-y: auto; font-size: 0.9em; }
    .label { font-weight: bold; margin-top: 20px; margin-bottom: 4px; }
    .section { background: #fff; border: 1px solid #ddd; padding: 16px; margin-bottom: 16px; }
    .section h2 { margin: 0 0 12px 0; font-size: 1em; color: #333; }
  </style>
</head>
<body>
  <h1>RAG Tool — nomic-embed-text + llama3.2:1b via Ollama</h1>

  <div class="section">
    <h2>Step 1 — Index your documents</h2>
    <div class="label">Documents folder path</div>
    <div class="row">
      <input type="text" id="docs_dir" placeholder="docs" value="docs" />
      <button onclick="call('load_docs', 'docs_dir')">Preview Docs</button>
      <button class="primary" onclick="call('index', 'docs_dir')">Embed &amp; Index</button>
      <button onclick="call('stats', null)">Index Stats</button>
    </div>
  </div>

  <div class="section">
    <h2>Step 2 — Query your documents</h2>
    <div class="label">Your question</div>
    <div class="row">
      <input type="text" id="query" placeholder="What is the return policy?" />
      <button onclick="call('retrieve', 'query')">Retrieve Chunks</button>
      <button class="primary" onclick="call('ask', 'query')">Ask (Full RAG)</button>
    </div>
  </div>

  <div class="label">Output</div>
  <pre id="output">Results will appear here...</pre>

  <script>
    async function call(action, inputId) {
      const value = inputId ? document.getElementById(inputId).value.trim() : null;
      document.getElementById('output').textContent = 'Working...';
      try {
        const body = {};
        if (action === 'index' || action === 'load_docs') body.docs_dir = value || 'docs';
        if (action === 'retrieve' || action === 'ask') body.query = value;

        const r = await fetch('/api/' + action, {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify(body)
        });
        const data = await r.json();
        document.getElementById('output').textContent =
          typeof data.result === 'object'
            ? JSON.stringify(data.result, null, 2)
            : data.result;
      } catch(e) {
        document.getElementById('output').textContent = 'Request failed: ' + e;
      }
    }
  </script>
</body>
</html>
"""

@app.route("/")
def index():
    return render_template_string(HTML)

@app.route("/api/load_docs", methods=["POST"])
def api_load_docs():
    docs_dir = request.json.get("docs_dir", DOCS_DIR)
    docs = tools.load_documents(docs_dir)
    summary = [{"filename": d["filename"], "preview": d["content"][:200]} for d in docs]
    return jsonify(result={"count": len(docs), "documents": summary})

@app.route("/api/index", methods=["POST"])
def api_index():
    docs_dir = request.json.get("docs_dir", DOCS_DIR)
    return jsonify(result=tools.embed_and_index(docs_dir))

@app.route("/api/stats", methods=["POST"])
def api_stats():
    return jsonify(result=tools.index_stats())

@app.route("/api/retrieve", methods=["POST"])
def api_retrieve():
    query = request.json.get("query", "")
    chunks = tools.retrieve_chunks(query)
    return jsonify(result=chunks)

@app.route("/api/ask", methods=["POST"])
def api_ask():
    query = request.json.get("query", "")
    return jsonify(result=tools.answer_with_context(query))

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run it:

```bash
python app.py
```

Then open: [http://localhost:5000](http://localhost:5000)

---

## Part 5 — Hands-On Exercises

Work through these in order using whichever interface you chose.

### Exercise 1 — Load and Preview Documents

CLI:

```
Choose: 1
```

Browser: click **Preview Docs** (leave folder as `docs`).

Expected: filenames listed, preview of the first document shown.

---

### Exercise 2 — Embed and Index

CLI:

```
Choose: 2
Docs folder (default: docs): docs
```

Browser: click **Embed & Index**.

Expected: confirmation of how many documents and chunks were indexed. An `index.json` file is created in your project folder.

> **Note:** Embedding takes a few seconds per chunk. A 5-document set typically finishes in under 30 seconds.

---

### Exercise 3 — Inspect the Index

CLI:

```
Choose: 3
```

Browser: click **Index Stats**.

Expected: total chunks, list of indexed filenames, and path to `index.json`.

---

### Exercise 4 — Retrieve Without Generation

Stage a question that should match your documents:

CLI:

```
Choose: 4
Query: What is the return window for electronics?
```

Browser: type a question, click **Retrieve Chunks**.

Expected: top 3 chunks with similarity scores and the source filenames. No LLM involved yet — pure vector search.

---

### Exercise 5 — Full RAG Query

CLI:

```
Choose: 5
Your question: What is the return window for electronics?
```

Browser: type a question, click **Ask (Full RAG)**.

Expected: a grounded answer from Ollama citing only the indexed documents, plus the source filenames used.

---

### Exercise 6 — Challenge: Unanswerable Question

Ask something that is NOT in your documents:

```
What is the capital of France?
```

Expected: the model should respond with "I don't have that information." rather than hallucinating an answer — because the context does not contain it.

---

### Exercise 7 — Full Pipeline (Challenge)

1. Add a new `.txt` file to `docs/`
2. Re-run **Embed & Index**
3. Ask a question that only the new file can answer
4. Verify the answer and source are correct

---

## Part 6 — Troubleshooting

### Ollama not responding

```bash
# Check it is running
curl http://localhost:11434/api/tags

# Start it if not
ollama serve

# Confirm both models are downloaded
ollama list
```

---

### Embedding model missing

```bash
ollama pull nomic-embed-text
ollama list
```

---

### No documents found

```bash
# Check your docs folder has .txt or .md files
ls -la docs/

# Test tools.py directly
python -c "import tools; print(tools.load_documents('docs'))"
```

---

### Index not found when querying

```bash
# Build the index first
python -c "import tools; print(tools.embed_and_index('docs'))"

# Confirm index.json was created
ls -la index.json
```

---

### Flask not found

```bash
pip install flask
```

---

### Empty or wrong answers

- Ensure the query language matches the document language
- Try lower `TOP_K` first, then increase if answers are missing context
- Check chunk sizes: if chunks are too small, context is lost; if too large, noise increases

---


