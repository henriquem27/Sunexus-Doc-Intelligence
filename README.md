# Sunexus Group — Document Intelligence System
**Internship Project | Henrique Rio | May–August 2026**

A lightweight RAG-based AI tool that reads Sunexus Group's internal OneDrive documents and allows staff to query them in plain language. Built to accelerate scope deliverables (KPI dictionary, governance docs, final report) and optionally handed off as a permanent internal tool.

---

## Stack

| Layer | Tool |
|---|---|
| File Ingestion | `python-docx`, `openpyxl`, `PyPDF2` |
| OneDrive Access | Microsoft Graph API |
| Embeddings | `sentence-transformers` |
| Vector Store | ChromaDB (local, no server needed) |
| LLM | Claude API (`claude-sonnet-4-20250514`) |
| Interface | Streamlit |

---

## Project Structure

```
sunexus-dis/
├── README.md
├── requirements.txt
├── .env.example
├── ingest/
│   ├── __init__.py
│   ├── onedrive.py        # Graph API connector
│   ├── parsers.py         # Word, Excel, PDF parsers
│   └── chunker.py         # Text splitting logic
├── embeddings/
│   ├── __init__.py
│   └── embed.py           # Sentence-transformer embedding pipeline
├── vectorstore/
│   ├── __init__.py
│   └── chroma.py          # ChromaDB read/write helpers
├── rag/
│   ├── __init__.py
│   ├── retriever.py       # Query → relevant chunks
│   └── generator.py       # Claude API call with retrieved context
├── app/
│   └── streamlit_app.py   # Chat UI
├── scripts/
│   ├── build_index.py     # One-time script: ingest + embed + store
│   └── update_index.py    # Re-index when new docs are added
├── docs/
│   ├── user_guide.md      # Plain-language guide for non-technical staff
│   └── add_documents.md   # How to add new files to the knowledge base
└── tests/
    ├── test_parsers.py
    └── test_retriever.py
```

---

## Setup

### 1. Clone and install dependencies

```bash
git clone https://github.com/yourhandle/sunexus-dis.git
cd sunexus-dis
pip install -r requirements.txt
```

### 2. Configure environment variables

Copy `.env.example` to `.env` and fill in:

```bash
cp .env.example .env
```

```env
ANTHROPIC_API_KEY=your_key_here
AZURE_CLIENT_ID=your_azure_app_id
AZURE_CLIENT_SECRET=your_azure_secret
AZURE_TENANT_ID=your_tenant_id
ONEDRIVE_FOLDER_PATH=/Sunexus/DIS-Documents
CHROMA_PERSIST_DIR=./vectorstore/chroma_db
```

### 3. Build the index

```bash
python scripts/build_index.py
```

This will:
- Connect to OneDrive via Graph API
- Download and parse all documents in the configured folder
- Embed each chunk using sentence-transformers
- Store everything in ChromaDB locally

### 4. Run the app

```bash
streamlit run app/streamlit_app.py
```

---

## Usage Examples

Once running, staff can ask questions like:

- *"What is our definition of occupancy rate for the Hollywood property?"*
- *"What data gaps exist in our current lead tracking process?"*
- *"Draft the update instructions for the executive dashboard."*
- *"Summarize our investor reporting requirements."*
- *"What is the check-in process for Cielo Azul-SG units?"*

The system only answers from the documents you have given it. It will say when it does not know something rather than making information up.

---

## Adding New Documents

1. Upload the file to the configured OneDrive folder
2. Run `python scripts/update_index.py`
3. The new document will be available immediately in the app

See `docs/add_documents.md` for a step-by-step guide for non-technical staff.

---

## TODO

### Phase I — Setup (Days 1–15)

- [ ] Register Azure app and configure Microsoft Graph API access for OneDrive
- [ ] Set up project repository and folder structure
- [ ] Write `.env.example` with all required variables documented
- [ ] Audit available OneDrive documents and identify which ones to ingest first
- [ ] Test Graph API connection and basic file download

### Phase II — Build (Days 16–30)

- [ ] Write parsers for Word (`.docx`), Excel (`.xlsx`), and PDF files
- [ ] Implement text chunking strategy (chunk size, overlap, metadata tagging)
- [ ] Set up ChromaDB and test local persistence
- [ ] Implement sentence-transformer embedding pipeline
- [ ] Write `build_index.py` script
- [ ] Test end-to-end: OneDrive file → parsed text → embedded chunks → stored in ChromaDB
- [ ] Write basic retriever: query → top-k relevant chunks
- [ ] Connect Claude API: retrieved chunks → grounded response
- [ ] Test retriever quality on KPI and business plan questions
- [ ] Build basic Streamlit chat interface (input box, response display, source references)

### Phase III — Validate & Finalize (Days 31–60)

- [ ] Validate system outputs against manually verified KPI definitions
- [ ] Use system to generate first draft of KPI Dictionary v1
- [ ] Use system to generate first draft of Business Understanding Memo
- [ ] Use system to generate draft of Data Gaps Report
- [ ] Tune chunking and retrieval if outputs are off
- [ ] Write `update_index.py` for adding new documents without rebuilding from scratch
- [ ] Add source citation display in Streamlit UI (which document answered the question)
- [ ] Add basic error handling (API timeouts, missing files, empty responses)
- [ ] Write `docs/user_guide.md` in plain language for non-technical staff
- [ ] Write `docs/add_documents.md`

### Phase IV — Handoff (Days 61–90)

- [ ] Write unit tests for parsers and retriever
- [ ] Final README review and cleanup
- [ ] Record a short walkthrough video for the team (optional but useful)
- [ ] Conduct walkthrough session with Carolina and any relevant staff
- [ ] Confirm the system runs correctly on company hardware or browser
- [ ] Add system outputs to Final Internship Report as documented deliverable
- [ ] Hand over repository, credentials documentation, and handoff package to Camilo

---

## Scope Deliverables This Project Supports

| Deliverable | How |
|---|---|
| KPI Dictionary v1 | System generates consistent metric definitions from internal documents |
| Business Understanding Memo | System summarizes business plan and operating files |
| Data Gaps Report | System identifies missing fields and inconsistencies |
| Dashboard Update Manual | System drafts update instructions from dashboard logic |
| Light Governance Framework | System generates ownership and cadence recommendations |
| Final Report | System synthesizes all 90-day work into a structured document |

---

## Notes

- The system never fabricates information. If relevant content is not in the knowledge base, it says so.
- Embeddings are generated locally using `sentence-transformers` — no data is sent to a third party during indexing.
- Only the query and retrieved chunks are sent to the Claude API at response time.
- ChromaDB persists on disk. Rebuilding the index is only necessary when documents change significantly.

---

*Built by Henrique Rio as part of the Sunexus Group Data & Analytics Internship, May–August 2026.*
