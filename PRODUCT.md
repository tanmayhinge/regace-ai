# Regace AI — Product & Architecture Overview

> RAG-powered regulatory intelligence for Indian pharma. Users ask questions about CDSCO, NPPA, and Drugs & Cosmetics Act regulations and get streamed answers with numbered citations linked to source document passages.

---

## How It Works (End-to-End)

```
User types a question
       |
       v
Frontend sends POST /api/chat (SSE stream)
       |
       v
Backend embeds the query (local model, no API call)
       |
       v
Hybrid retrieval: ChromaDB cosine similarity + BM25 keyword search
       |
       v
Reciprocal Rank Fusion merges rankings
       |
       v
Query-hint detection: if user mentions "Rule 122-A" or "Schedule M",
a targeted metadata filter boosts exact matches
       |
       v
Parent expansion: fetches sibling chunks from the same regulatory
section so Claude sees the full Rule/Section, not just a fragment
       |
       v
SSE event: "sources" — frontend renders source cards immediately
       |
       v
Chunks injected into Claude's system prompt with hierarchy breadcrumbs
Claude streams an answer with [1], [2] citations
       |
       v
SSE events: "token" (repeated) — frontend renders text in real-time
       |
       v
SSE event: "done" — frontend filters sources to only cited ones
       |
       v
Chat + messages persisted to SQLite
```

---

## What Runs Locally vs. Cloud

| Component | Where | Notes |
|-----------|-------|-------|
| Embeddings (`all-MiniLM-L6-v2`) | Local | No API key needed, runs on CPU |
| Vector DB (ChromaDB) | Local | Persists to `backend/data/chroma_db/` |
| BM25 keyword index | Local | Built in-memory at server startup |
| Chat history (SQLite) | Local | `backend/data/regace.db` |
| PDF storage | Local | `backend/data/pdfs/` |
| **Answer generation (Claude)** | **Cloud** | **Only external call — requires `ANTHROPIC_API_KEY`** |

---

## Tech Stack

- **Frontend**: Next.js 14 (App Router) + TypeScript + Tailwind CSS
- **Backend**: Python 3.11 + FastAPI + Uvicorn
- **Vector DB**: ChromaDB (embedded, cosine similarity)
- **Keyword Search**: rank-bm25 (BM25Okapi)
- **Embeddings**: sentence-transformers (`all-MiniLM-L6-v2`, 384 dimensions)
- **LLM**: Claude Sonnet 4 via Anthropic API (streaming SSE)
- **Chat Persistence**: SQLite with WAL mode
- **PDF Extraction**: pdfplumber (+ optional pytesseract OCR fallback)

---

## Frontend Architecture

### Layout (Desktop)

Three-column layout using `react-resizable-panels`:

```
|  Sidebar  |       Chat Panel       |  Sources Panel  |
|  (fixed)  |      (resizable)       |   (resizable)   |
|  260px    |         ~60%           |      ~40%       |
|  or 52px  |                        |                 |
| collapsed |                        |                 |
```

- Sidebar has a fixed width — either 260px (expanded) or 52px (collapsed). Not resizable.
- Chat and Sources panels are resizable relative to each other via a drag handle.
- Layout state auto-saved to localStorage.

### Layout (Mobile < 1024px)

Tab bar at top: **Chats** | **Chat** | **Sources**

Each tab shows a full-screen view. Clicking a citation in Chat auto-switches to Sources tab.

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| `ChatSidebar` | `chat-sidebar.tsx` | Chat list, new inquiry button, collapse toggle |
| `ChatPanel` | `chat-panel.tsx` | Message list, empty state with example queries, chat input |
| `MessageBubble` | `message-bubble.tsx` | Renders a single message, parses `[N]` citations into clickable links |
| `CitationLink` | `citation-link.tsx` | Amber `[N]` badge — clicks scroll to and highlight the matching source card |
| `SourcesPanel` | `sources-panel.tsx` | Scrollable list of source cards |
| `SourceCard` | `source-card.tsx` | Displays passage text, category badge, hierarchy breadcrumb, page range |
| `ChatInput` | `chat-input.tsx` | Auto-expanding textarea, Enter to send, Shift+Enter for newline |
| `PdfViewerModal` | `pdf-viewer-modal.tsx` | Overlay modal with iframe PDF viewer, navigates to correct page |

### State Management

All chat state lives in the `useChat` hook (`src/hooks/use-chat.ts`):

```typescript
chatList: ChatListItem[]           // All chats
activeChatId: string | null        // Selected chat
messages: Message[]                // Messages in active chat
sources: Source[]                  // Retrieved sources for latest query
isStreaming: boolean               // True while Claude is generating
error: string | null               // Error display
highlightedSource: number | null   // Which source card to highlight (citation click)
```

After streaming completes, sources are filtered to only those actually cited in the response (falls back to first 5 if no citations detected).

### Design System

| Element | Value |
|---------|-------|
| Display font | Instrument Serif (headings) |
| Body font | DM Sans (UI text) |
| Background | Warm cream (`hsl(40 33% 98%)`) |
| Sidebar | Navy-950 (`#1e1b2f`) with noise texture overlay |
| Accent | Amber/gold for citations, highlights, active states |
| Category colors | Rules=amber, Guidelines=emerald, Notification=sky, Circular=violet, Order=rose |

Logo variants:
- `regace-logo.png` — white, for dark sidebar
- `regace-logo-black.png` — dark, for chat empty state
- `regace-logo-small.png` — icon, for collapsed sidebar + message avatars

---

## Backend Architecture

### API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/chat` | Main endpoint — SSE stream of sources + Claude tokens |
| `GET` | `/api/chats` | List all chats (sorted by most recent) |
| `POST` | `/api/chats` | Create a new chat |
| `GET` | `/api/chats/{id}` | Get chat with all messages and sources |
| `DELETE` | `/api/chats/{id}` | Delete chat and all its messages |
| `GET` | `/health` | Health check |
| `GET` | `/api/pdfs/{filename}` | Serve PDF files for the viewer modal |

### SSE Event Protocol

The `/api/chat` endpoint returns a Server-Sent Events stream:

```
1. event: sources
   data: {"sources": [...], "chat_id": "abc123"}
   → Sent immediately after retrieval. Frontend renders source cards.

2. event: token  (repeated many times)
   data: {"token": "The "}
   data: {"token": "regulation "}
   data: {"token": "states..."}
   → Each text fragment from Claude. Frontend appends to message.

3. event: done
   data: {}
   → Stream complete. Frontend filters sources to cited ones.
```

### Ingestion Pipeline

```
PDF file
  → pdfplumber extracts text from every page
  → All pages concatenated into single document with <<PAGE:N>> markers
  → Hierarchy index built (Schedule > Part > Chapter > Section > Rule)
  → Section-aware chunking on full text (~1000 chars, 200 char overlap)
  → Each chunk gets: hierarchy breadcrumb, parent_id, page range, section metadata
  → Embeddings generated locally (all-MiniLM-L6-v2)
  → Stored in ChromaDB with all metadata
```

Key improvement: **cross-page assembly** — sections that span multiple pages stay intact instead of being split at page boundaries.

Chunk metadata stored in ChromaDB:
```
title, date, category, document_type, page, page_end,
source_file, hierarchy, parent_id, chunk_position,
section_number, rule_number, schedule, form_number
```

### Retrieval Pipeline

```
User query
  │
  ├─ Embed query → ChromaDB cosine similarity (top 75)
  ├─ BM25 keyword search (top 75)
  ├─ Query-hint filter: detect "Rule 122-A" etc. → metadata lookup
  │
  └─ Reciprocal Rank Fusion merges all ranked lists
       │
       └─ Top 25 chunks selected
            │
            └─ Parent expansion: fetch sibling chunks (same parent_id)
                 Max 5 siblings per parent, 30 total chunks cap
                 Sorted by document position for coherent reading
```

### Context Formatting (What Claude Sees)

```
[1] Drugs & Cosmetics Rules, 1945 | Schedule M > Part I > Section 5
    Rules | 1945-04-01 | Pages 45-47
In exercise of the powers conferred by section 12 of the Drugs and
Cosmetics Act, 1940...

[2] NDCT Rules, 2019 | Chapter II > Rule 7
    Rules | 2019-03-19 | Pages 12-13
The applicant shall submit an application in Form CT-04...
```

The system prompt enforces:
- Every claim must cite `[1]`, `[2]`, etc.
- Only use information from provided sources (no prior knowledge)
- Never fabricate regulation numbers or dates
- Quote regulatory text verbatim when possible
- Use hierarchy context to understand document structure

---

## Data Types

### Source (Frontend)
```typescript
interface Source {
  id: string;
  index: number;          // Citation number [1], [2], etc.
  title: string;          // Document title
  date: string;           // "2019-03-19" or "Unknown"
  category: string;       // "Rules", "Notification", etc.
  document_type?: string; // "Act", "Rules", "Guideline", etc.
  page: number;           // Start page
  page_end?: number;      // End page (if spans multiple)
  hierarchy?: string;     // "Schedule M > Part I > Section 5"
  text: string;           // Passage content
  score: number;          // Retrieval relevance score
  source_file?: string;   // PDF filename
}
```

### Message
```typescript
interface Message {
  role: "user" | "assistant";
  content: string;
  sources?: Source[];     // Attached to assistant messages
}
```

---

## Database Schema (SQLite)

```sql
chats
  id          TEXT PRIMARY KEY
  title       TEXT
  created_at  TEXT (ISO timestamp)
  updated_at  TEXT (ISO timestamp)

messages
  id           TEXT PRIMARY KEY
  chat_id      TEXT → chats.id (CASCADE delete)
  role         TEXT ('user' | 'assistant')
  content      TEXT
  sources_json TEXT (JSON array of Source objects)
  created_at   TEXT (ISO timestamp)
```

---

## MVP Knowledge Base

8 target documents covering ~90% of daily regulatory affairs queries:

1. **Drugs & Cosmetics Act, 1940** — foundational statute
2. **Drugs & Cosmetics Rules, 1945** — operational rules (Rule 122A, 122E, 74, etc.)
3. **New Drugs and Clinical Trials Rules, 2019** — new drug approvals, clinical trials
4. **DPCO 2013** — drug pricing, ceiling price formula
5. **Revised Schedule M (2024)** — GMP requirements
6. **Cosmetics Rules, 2020** — cosmetics regulatory track
7. **Medical Devices Rules, 2017** — device classification and licensing
8. **CDSCO FDC Guidelines** — fixed-dose combination approvals

---

## Running the Project

### Prerequisites
- Python 3.11+
- Node.js 18+
- `ANTHROPIC_API_KEY` in `backend/.env`

### Backend
```bash
cd backend
source .venv/bin/activate
uvicorn app.main:app --reload --port 8000
```

### Frontend
```bash
cd frontend
npm run dev
```

### Ingest Documents
```bash
cd backend
source .venv/bin/activate
python -m scripts.ingest --reset --pdf-dir ./data/pdfs
```
After ingesting, restart the backend to rebuild the BM25 index.

### Docker
```bash
docker-compose up
# Backend: http://localhost:8000
# Frontend: http://localhost:3000
```

---

## Environment Variables

| Variable | Location | Default | Required |
|----------|----------|---------|----------|
| `ANTHROPIC_API_KEY` | `backend/.env` | — | Yes |
| `ANTHROPIC_MODEL` | `backend/.env` | `claude-sonnet-4-20250514` | No |
| `CHROMA_DB_PATH` | `backend/.env` | `./data/chroma_db` | No |
| `EMBEDDING_MODEL` | `backend/.env` | `all-MiniLM-L6-v2` | No |
| `TOP_K` | `backend/.env` | `25` | No |
| `FRONTEND_URL` | `backend/.env` | `http://localhost:3000` | No |
| `NEXT_PUBLIC_API_URL` | `frontend/.env.local` | `http://localhost:8000` | No |
