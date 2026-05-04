# Pilot — Local Architecture

Simple local variant for validating the pipeline with a small document set and no cloud dependencies.

## What changes

| Component | Main stack | Pilot |
|---|---|---|
| Document source | OneDrive (Graph API) | Local folder (`pilot/docs/`) |
| LLM | Claude API | Ollama (`llama3.2`) |
| Embeddings | `sentence-transformers` | unchanged |
| Vector store | ChromaDB | unchanged |
| Interface | Streamlit | unchanged |

No Azure credentials or `ANTHROPIC_API_KEY` needed.

## Setup

### 1. Install Ollama and pull the model

```bash
# macOS
brew install ollama
ollama serve          # starts the local server on http://localhost:11434
ollama pull llama3.2  # ~2 GB, good balance of speed and quality
```

### 2. Install Python dependencies

```bash
pip install -r requirements.txt
pip install ollama    # Ollama Python client
```

### 3. Add documents

Drop `.docx`, `.xlsx`, or `.pdf` files into `pilot/docs/`. A handful of representative documents is enough to validate chunking, retrieval, and generation quality.

### 4. Build the pilot index

```bash
python scripts/pilot_build_index.py
```

### 5. Run the app

```bash
streamlit run app/streamlit_app.py
```

## File structure

```
pilot/
├── docs/                  # Drop test documents here
└── chroma_db/             # Pilot vector store (separate from production)
scripts/
└── pilot_build_index.py   # Reads from pilot/docs/ instead of OneDrive
```

## Code differences

**`scripts/pilot_build_index.py`** — replaces OneDrive download with a local directory walk:

```python
from pathlib import Path
from ingest.parsers import parse_file
from ingest.chunker import chunk_text
from embeddings.embed import embed_chunks
from vectorstore.chroma import store_chunks

DOCS_DIR = Path("pilot/docs")
CHROMA_DIR = "pilot/chroma_db"

for path in DOCS_DIR.iterdir():
    text = parse_file(path)
    chunks = chunk_text(text, source=path.name)
    embeddings = embed_chunks(chunks)
    store_chunks(chunks, embeddings, persist_dir=CHROMA_DIR)
```

**`rag/generator.py`** — swap the Claude API call for Ollama:

```python
import ollama

def generate(query: str, context_chunks: list[str]) -> str:
    context = "\n\n".join(context_chunks)
    prompt = (
        f"Answer using only the context below. "
        f"If the answer is not in the context, say so.\n\n"
        f"Context:\n{context}\n\nQuestion: {query}"
    )
    response = ollama.chat(
        model="llama3.2",
        messages=[{"role": "user", "content": prompt}],
    )
    return response["message"]["content"]
```

## Environment variables for pilot

Only two are needed:

```env
CHROMA_PERSIST_DIR=pilot/chroma_db
```

Ollama runs on `http://localhost:11434` by default — no key required.

## Validation checklist

- [ ] Parsers correctly extract text from each file type in `pilot/docs/`
- [ ] Chunks look reasonable (not too short/long) — inspect with `chromadb` CLI or a quick script
- [ ] Retriever returns relevant chunks for a sample query
- [ ] Llama response is grounded in the document content and declines when the answer isn't there
- [ ] Response latency is acceptable for the intended hardware
