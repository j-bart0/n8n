# n8n Workflows

A collection of personal n8n automation workflows for music production and label management.

## Workflows

### 1. `ableton-control.json` — DAW Session Creator
Creates Ableton Live sessions via natural language through an AI agent.

**Stack:** n8n Chat Trigger → Ollama (local LLM) → JavaScript parser → Mac Bridge (local Flask server)

**How it works:**
- User describes a session in natural language ("un set rock con kick, snare, due chitarre e voce")
- Ollama parses the description into a structured JSON
- A JS node maps instruments to Ableton track types, colors, routing, and I/O
- A local Mac Bridge Flask server receives the JSON and creates the session in Ableton via AbletonMCP

**Requirements:**
- Ollama running locally with a chat model
- [AbletonMCP](https://github.com/...) or compatible Mac Bridge server on port `5679`
- n8n with `@n8n/n8n-nodes-langchain` package

**Credentials to configure:**
- `YOUR_OLLAMA_CREDENTIAL_ID` → Ollama API (local, no key needed)
- `YOUR_OLLAMA_MODEL` → e.g. `qwen2.5:7b`, `llama3.1:8b`

---

### 2. `mail-sender.json` — AI Email Agent
Sends emails to contacts stored in a Notion database using natural language.

**Stack:** n8n Chat Trigger → Ollama AI Agent → Notion DB lookup → SMTP send

**How it works:**
- User says "send an email to [Artist Name] telling them [message]"
- Ollama rewrites it as a professional email (in the user's language)
- The workflow looks up the contact's email address in a Notion "Artists" database
- Sends the email via SMTP

**Notion DB schema required:**
- `Artists` database with properties: `Nome` (title), `Artista` (text), `Email` (email)

**Credentials to configure:**
- `YOUR_SMTP_CREDENTIAL_ID` → SMTP account (Gmail App Password or other)
- `YOUR_NOTION_CREDENTIAL_ID` → Notion Integration token
- `YOUR_NOTION_ARTISTS_DB_ID` → Notion Database ID for contacts
- `YOUR_OLLAMA_CREDENTIAL_ID` / `YOUR_OLLAMA_MODEL` → local LLM
- `YOUR_EMAIL@example.com` → sender email address

---

### 3. `rag.json` — Local RAG (Retrieval-Augmented Generation)
A dual-function workflow: uploads documents to a Qdrant vector store AND answers questions using them.

**Stack:** 
- Upload path: n8n Form Trigger → Document Loader → Qdrant (insert) + Ollama Embeddings
- Query path: n8n Chat Trigger → Ollama AI Agent → Qdrant (retrieve-as-tool) + Redis Memory

**How it works:**
- A web form allows uploading files (PDF, DOCX, MD, CSV, images)
- Files are chunked, embedded via `nomic-embed-text` (Ollama), and stored in Qdrant
- The chat interface queries the vector store and answers in the user's language
- Redis maintains conversation memory across sessions

**Requirements:**
- Ollama with `nomic-embed-text` and a chat model
- Qdrant running locally (default: `localhost:6333`)
- Redis running locally (default: `localhost:6379`)

**Credentials to configure:**
- `YOUR_OLLAMA_CREDENTIAL_ID` / `YOUR_OLLAMA_MODEL`
- `YOUR_QDRANT_CREDENTIAL_ID` → Qdrant API (local, no key if no auth)
- `YOUR_QDRANT_COLLECTION_NAME` → e.g. `kb_wiki`, `knowledge_base`
- `YOUR_REDIS_CREDENTIAL_ID` → Redis connection

---

### 4. `royalties.json` — Soundrop Royalties Tracker
Automatically parses Soundrop CSV statements from Gmail and writes cumulative royalty data to Notion.

**Stack:** Cron → Gmail (attachment download) → ZIP decompress → CSV parse → JS aggregation → Notion CRUD

**How it works:**
- Runs daily; searches Gmail for unprocessed Soundrop statement emails with attachments
- Downloads the ZIP attachment, extracts the CSV
- Aggregates streams and revenue per ISRC across services
- Checks Notion `Royalties` DB: updates existing entries or creates new ones
- Calculates cumulative totals, period delta %, and labels processed emails

**Notion DB schemas required:**
- `Royalties`: `ISRC` (text), `Total Streams` (number), `Total Revenue USD` (number), `Last Period Revenue` (number), `Prev Period Revenue` (number), `Delta %` (number), `Last Statement Period` (text), `Service` (text)
- `Releases`: `UPC` (text) — for release lookup
- `Artists`: `Artista` (title) — for artist lookup

**Credentials to configure:**
- `YOUR_GMAIL_CREDENTIAL_ID` → Gmail OAuth2
- `YOUR_GMAIL_LABEL_ID` → Gmail label ID for processed emails (find it via Gmail API or URL)
- `YOUR_GMAIL_LABEL_NAME` → label name used in the Gmail search filter
- `YOUR_LABEL_NAME` → your label/distributor name as it appears in email subjects
- `YOUR_NOTION_CREDENTIAL_ID`
- `YOUR_NOTION_ROYALTIES_DB_ID`
- `YOUR_NOTION_RELEASES_DB_ID`
- `YOUR_NOTION_ARTISTS_DB_ID`

---

### 5. `song-metadata.json` — Audio Metadata Generator
Polls a folder for new audio files, analyzes them with a local service, and writes metadata to Notion.

**Stack:** Cron (1 min) → HTTP (audio-analyzer service) → Notion DB diff → analyze → Notion create

**How it works:**
- Every minute, calls a local `audio-analyzer` microservice to list audio files in a watched folder
- Compares the list against existing entries in a Notion `Songs` database
- New files are sent to the analyzer (Python/librosa-based) which returns BPM, key, mood, duration, sample rate, bit depth
- Results are written as new pages in the Notion `Songs` DB

**Requirements:**
- `audio-analyzer` Docker service on `http://audio-analyzer:8001` (must expose `/list` and `/analyze` endpoints)
- Audio files accessible at `/audio/` inside the Docker network

**Notion DB schema required:**
- `Songs`: `BPM` (number), `Description` (text), `Duration` (text), `Key` (select), `Mood` (multi-select), `Sample Rate` (number), `Bit-depth` (text)

**Credentials to configure:**
- `YOUR_NOTION_CREDENTIAL_ID`
- `YOUR_NOTION_SONGS_DB_ID`

---

## General Setup Notes

1. All workflows have `"active": false` — enable them manually after configuring credentials.
2. Replace all `YOUR_*` placeholders before importing.
3. Node IDs (`node-001`, etc.) are placeholders — n8n will reassign them on import.
4. The `instanceId` in `meta` is a placeholder; n8n will use your own instance ID.

## Architecture Overview

```
Local Stack:
  Ollama          → local LLM inference (chat + embeddings)
  Qdrant          → vector database
  Redis           → conversation memory
  Mac Bridge      → Flask server bridging n8n → Ableton
  audio-analyzer  → Python/librosa microservice

External:
  Notion          → persistent data store (Songs, Artists, Releases, Royalties)
  Gmail           → source for Soundrop statements
  SMTP            → email delivery
  Soundrop        → music distributor (statement source)
```
