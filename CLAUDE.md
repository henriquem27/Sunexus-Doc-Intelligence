# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A RAG-based document intelligence system for Sunexus Group. Reads internal OneDrive documents (Word, Excel, PDF) via Microsoft Graph API, embeds them locally with `sentence-transformers`, stores in ChromaDB, and answers staff queries through a Streamlit chat UI backed by the Claude API.

## Stack

- **File ingestion**: `python-docx`, `openpyxl`, `PyPDF2`
- **OneDrive access**: Microsoft Graph API (Azure app credentials)
- **Embeddings**: `sentence-transformers` (runs locally — no data leaves the machine during indexing)
- **Vector store**: ChromaDB (local, file-based persistence at `CHROMA_PERSIST_DIR`)
- **LLM**: Claude API (`claude-sonnet-4-20250514`)
- **Interface**: Streamlit

## Environment Setup

```bash
pip install -r requirements.txt
cp .env.example .env  # fill in ANTHROPIC_API_KEY, AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID, ONEDRIVE_FOLDER_PATH, CHROMA_PERSIST_DIR
```

## Commands

```bash
# Build the vector index (run once, or after significant document changes)
python scripts/build_index.py

# Update index with newly added OneDrive documents (no full rebuild needed)
python scripts/update_index.py

# Run the Streamlit chat app
streamlit run app/streamlit_app.py

# Run tests
pytest tests/
pytest tests/test_parsers.py   # single test file
pytest tests/test_retriever.py
```

## Architecture

Data flows in two phases:

**Indexing** (`scripts/build_index.py` / `scripts/update_index.py`):
`OneDrive (Graph API)` → `ingest/onedrive.py` → `ingest/parsers.py` → `ingest/chunker.py` → `embeddings/embed.py` → `vectorstore/chroma.py`

**Query** (`app/streamlit_app.py`):
`User query` → `rag/retriever.py` (ChromaDB similarity search) → `rag/generator.py` (Claude API with retrieved context) → `Streamlit response`

### Key modules

- `ingest/onedrive.py` — Microsoft Graph API connector; downloads files from the configured OneDrive folder
- `ingest/parsers.py` — format-specific text extractors for `.docx`, `.xlsx`, `.pdf`
- `ingest/chunker.py` — text splitting with configurable chunk size, overlap, and metadata tagging
- `embeddings/embed.py` — sentence-transformer embedding pipeline
- `vectorstore/chroma.py` — ChromaDB read/write helpers; persistence path from `CHROMA_PERSIST_DIR` env var
- `rag/retriever.py` — converts a query to a top-k chunk list via ChromaDB similarity search
- `rag/generator.py` — assembles the Claude API prompt from retrieved chunks and returns a grounded response
- `app/streamlit_app.py` — chat UI with source citation display

## Design Constraints

- The system must only answer from indexed documents — never fabricate. `generator.py` must instruct the model to say when it doesn't know.
- Embeddings are generated locally; only the user query and retrieved chunks reach the Claude API.
- ChromaDB persists on disk; `build_index.py` is destructive (full rebuild). `update_index.py` is additive.
