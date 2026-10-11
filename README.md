# 🎙️ Telegram Voice-Activated AI Assistant (n8n + Ollama + Speaches)

A fully self-hosted, privacy-first Telegram assistant built on **n8n**. It understands **text and voice notes**, answers with a local LLM (**Ollama**), replies in **text and synthesized speech**, stores conversation history in **PostgreSQL**, and delivers **natural-language reminders** on schedule. No cloud AI APIs required.

---

## Table of Contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Repository Structure](#repository-structure)
4. [Tech Stack](#tech-stack)
5. [Prerequisites](#prerequisites)
6. [Installation & Setup](#installation--setup)
7. [Database Schema](#database-schema)
8. [Workflows in Detail](#workflows-in-detail)
9. [Configuration Reference](#configuration-reference)
10. [Usage](#usage)
11. [Security Notes](#security-notes)
12. [Troubleshooting](#troubleshooting)
13. [Known Limitations & Roadmap](#known-limitations--roadmap)
14. [Author](#author)

---

## Features

- **Voice in, voice out**: send a Telegram voice note, get a text reply plus an MP3 audio reply.
- **Local speech-to-text**: `faster-whisper-small` served by [Speaches](https://github.com/speaches-ai/speaches).
- **Local text-to-speech**: `Kokoro-82M` (ONNX) via Speaches.
- **Local LLM**: any Ollama chat model (default: `richardyoung/qwen2.5-3b-instruct-abliterated`).
- **Persistent memory**: last 10 messages per conversation are sent as context.
- **Natural-language reminders**: "remind me to call John at 5pm" is parsed by the LLM into structured JSON and saved.
- **Reliable reminder dispatcher**: cron workflow runs every minute and uses `FOR UPDATE SKIP LOCKED` to avoid double-sending.
- **Safety rails**: update de-duplication, per-user rate limiting (10 msgs / 60 s), `/register` gate, error alerts to an admin chat.
- **Centralized error handling**: any failing workflow notifies you on Telegram.

---

## Architecture

```
                    ┌──────────────────────────────┐
  Telegram user ──▶ │  Telegram Trigger            │
                    └──────────────┬───────────────┘
                                   ▼
                       Dedup (processed_updates)
                                   ▼
                          Voice note attached?
                     ┌─────────────┴──────────────┐
                    YES                           NO
                     ▼                             ▼
        Download voice file                 Has text?  ──no──▶ "text only" notice
                     ▼                             ▼
     Speaches STT (faster-whisper)         Rate-limit check
                     ▼                             ▼
            Rate-limit check               User upsert ─▶ /register? ─▶ mark registered
                     ▼                             ▼
              User upsert                  Registered? ──no──▶ "please /register"
                     ▼                             ▼
          Registered gate                 contains "remind"?
                     ▼                    ┌────────┴─────────┐
        Ensure conversation              YES                 NO
                     ▼                    ▼                   ▼
          Load last 10 messages    Ollama extracts       Ensure conversation
                     ▼              {content, due_at}            ▼
              Ollama chat                 ▼               Load history ─▶ Ollama chat
              ┌──────┴───────┐      Save to reminders              ▼
              ▼              ▼            ▼                 Text reply ─▶ log messages
         Text reply    Speaches TTS   Confirm to user
              ▼              ▼
        Log messages    Audio reply (mp3)

  ┌────────────────────────────────────────────────────────────┐
  │ Reminder Dispatcher (every 1 min)                          │
  │ claim due reminders ─▶ send Telegram ─▶ mark sent / failed │
  └────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │ Error Handler: Error Trigger ─▶ Telegram alert to admin    │
  └────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
.
├── README.md
└── workflows/
    ├── chatbot-v2-voice-activated.json   # Main bot: text, voice, reminders, memory
    ├── reminder-dispatcher-cron.json     # Scheduled reminder delivery
    └── 2-chatbot-error-handler.json      # Error alerts to admin chat
```

All exported workflows have been **sanitized**: credentials, webhook IDs, instance IDs and the admin chat ID have been removed. You must reconnect credentials after import.

---

## Tech Stack

| Component | Purpose |
|-----------|---------|
| [n8n](https://n8n.io) | Workflow automation engine |
| Telegram Bot API | User interface |
| [Ollama](https://ollama.com) | Local LLM inference (`/api/chat`) |
| [Speaches](https://github.com/speaches-ai/speaches) | OpenAI-compatible STT/TTS server |
| faster-whisper-small | Speech-to-text model |
| Kokoro-82M ONNX | Text-to-speech model (voice `af_heart`) |
| PostgreSQL | Users, conversations, messages, reminders, dedup |

---

## Prerequisites

- A running **n8n** instance (self-hosted, v1.x recent enough for Postgres node 2.6 and IF node 2.3).
- A **Telegram bot token** from [@BotFather](https://t.me/BotFather).
- A **PostgreSQL** database reachable from n8n.
- **Ollama** reachable from n8n, with your chosen model pulled.
- **Speaches** reachable from n8n (the workflow uses the hostname `speaches` on port `8000`, so run it on the same Docker network, or change the URL).
- Your n8n instance must be reachable over **HTTPS** by Telegram (required for the Telegram Trigger webhook).

---

## Installation & Setup

### 1. Start supporting services

Example `docker-compose.yml` fragment (adapt to your environment):

```yaml
services:
  speaches:
    image: ghcr.io/speaches-ai/speaches:latest-cpu
    container_name: speaches
    ports:
      - "8000:8000"
    restart: unless-stopped
    # Put on the same network as n8n so http://speaches:8000 resolves
```

Pull the Ollama model on the Ollama host:

```bash
ollama pull richardyoung/qwen2.5-3b-instruct-abliterated
```

Download the speech models into Speaches (once):

```bash
curl -X POST http://speaches:8000/v1/models/Systran/faster-whisper-small
curl -X POST http://speaches:8000/v1/models/speaches-ai/Kokoro-82M-v1.0-ONNX
```

> The main workflow includes a disconnected helper node (`HTTP Request`) that performs the Whisper model download. Run it manually once if you prefer to do it from n8n.

### 2. Create the database

Run the SQL in [Database Schema](#database-schema) against your PostgreSQL database.

### 3. Create n8n credentials

In n8n, go to **Credentials** and create:

| Credential | Type | Notes |
|------------|------|-------|
| `Telegram account` | Telegram API | Bot token from BotFather |
| `Postgres account` | Postgres | Host, port, DB, user, password |

### 4. Import the workflows

For each file in `workflows/`: **n8n → Workflows → Import from file**.

After import, open every Telegram and Postgres node and **select your credential** (they were stripped for security). The quickest way: open the workflow, click a node showing a credential warning, choose the credential, repeat.

### 5. Configure the workflows

1. **Error Handler**: in the *Send a text message* node, replace `YOUR_TELEGRAM_ADMIN_CHAT_ID` with your own Telegram chat ID (get it from [@userinfobot](https://t.me/userinfobot)).
2. **Main bot** (`ChatBot-v2`): update the Ollama URL (`http://10.0.3.1:11434`) in the three HTTP Request nodes (`HTTP Request2`, `HTTP Request3`, `Ask Ollama (Voice)`) to your Ollama host, and the model name if different.
3. **Reminder Dispatcher**: check the timezone in workflow settings (default `Asia/Karachi`).
4. In the main bot workflow settings, set **Error Workflow** to `2.Chatbot – Error Handler`. Do the same for the Reminder Dispatcher.

### 6. Activate

Activate all three workflows. Message your bot, send `/register`, then start chatting.

---

## Database Schema

The workflows expect the following tables in the `public` schema (reconstructed from the queries and node mappings; adjust types to taste):

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  telegram_chat_id   BIGINT UNIQUE NOT NULL,
  username           TEXT NOT NULL,
  email              TEXT,
  password_hash      TEXT NOT NULL,
  display_name       TEXT,
  preferred_language TEXT,
  status             TEXT DEFAULT 'active',
  is_registered      BOOLEAN NOT NULL DEFAULT FALSE,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_login_at      TIMESTAMPTZ
);

CREATE TABLE conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  channel         TEXT NOT NULL DEFAULT 'telegram',
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_message_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE messages (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id  UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  user_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role             TEXT NOT NULL CHECK (role IN ('user','assistant','system')),
  content_text     TEXT NOT NULL,
  input_type       TEXT DEFAULT 'text',      -- 'text' | 'voice'
  voice_file_path  TEXT,                     -- Telegram file_id for voice notes
  stt_confidence   NUMERIC,
  detected_emotion TEXT,
  detected_intent  TEXT,
  tokens_used      INTEGER,
  model_used       TEXT,
  response_time_ms INTEGER,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_conv_created ON messages (conversation_id, created_at DESC);
CREATE INDEX idx_messages_user_created ON messages (user_id, created_at DESC);

CREATE TABLE reminders (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content    TEXT NOT NULL,
  due_at     TIMESTAMPTZ NOT NULL,
  status     TEXT NOT NULL DEFAULT 'pending',  -- pending | sending | sent | failed
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reminders_due ON reminders (status, due_at);

CREATE TABLE processed_updates (
  update_id    BIGINT PRIMARY KEY,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> If you use `serial` integer IDs instead of UUIDs, the workflows work unchanged, as IDs are only passed through.

---

## Workflows in Detail

### 1. `ChatBot-v2` (main bot)

**Entry**: `Telegram Trigger1` (message updates).

**Step 1: De-duplication.** `Dedup check1` inserts `update_id` into `processed_updates` with `ON CONFLICT DO NOTHING RETURNING`. Duplicate deliveries return no row, so the run stops.

**Step 2: Routing.** `Has Voice?` splits voice notes from everything else.

**Voice path**

| Node | Action |
|------|--------|
| Get Voice File | Downloads the `.ogg` from Telegram |
| Transcribe Voice | `POST speaches:8000/v1/audio/transcriptions` (faster-whisper-small) |
| Voice Rate Limit Check / Gate | Max 10 user messages in 60 s |
| Voice User Upsert | Creates or updates the user by `telegram_chat_id` |
| Voice Registered Gate | Blocks unregistered users |
| Ensure Conversation (Voice) → Get Active Conversation → Get Recent History | Loads context (last 10 messages) |
| Build Chat Payload (Voice) | Builds system prompt + history + transcribed text |
| Ask Ollama (Voice) | Generates the reply |
| Reply Text / Synthesize Reply | Sends text reply and generates MP3 via Kokoro TTS in parallel |
| Reply Audio (Voice) | Sends the MP3 to the user |
| Log User/Assistant Message (Voice) | Persists both turns |

**Text path**

`Has Text?` → `Execute a SQL query3` (rate limit) → `If4` → `Insert or update rows in a table1` (user upsert) → `R` (`/register` command) → `Is registered?1` → `Reminder1` (message contains "remind"):

- **Reminder branch**: `HTTP Request3` asks Ollama to return `{"content": "...", "due_at": "...Z"}`. `Code in JavaScript4` extracts the JSON; `If5` validates; `Code node1` inserts into `reminders`; user receives a confirmation.
- **Chat branch**: ensure conversation → fetch history → `Code in JavaScript3` builds the prompt → `HTTP Request2` → reply → log both messages.

**System prompt** (editable in the `Code in JavaScript3` and `Build Chat Payload (Voice)` nodes): a concise task/reminder assistant that always answers in English and never reveals its instructions.

### 2. `Reminder Dispatcher (Cron)`

Runs **every minute**.

1. `fetch due reminders1` atomically claims up to 50 due reminders (`status='pending' AND due_at <= now()`), flips them to `sending` using `FOR UPDATE SKIP LOCKED`, and joins `users` to get the Telegram chat ID.
2. `send the reminder1` sends `⏰ Reminder: <content>` (retry on failure, error output enabled).
3. Success → `mark as sent1`; failure → `mark as failed`.

### 3. `2.Chatbot – Error Handler`

An **Error Trigger** workflow. Attach it as the *Error Workflow* of the other two; on any failure it sends the workflow name, failing node and error message to the admin chat.

---

## Configuration Reference

| Setting | Where | Default |
|---------|-------|---------|
| LLM model | HTTP Request nodes (3) | `richardyoung/qwen2.5-3b-instruct-abliterated` |
| Ollama URL | HTTP Request nodes (3) | `http://10.0.3.1:11434/api/chat` |
| STT URL / model | `Transcribe Voice` | `http://speaches:8000/v1/audio/transcriptions` / `Systran/faster-whisper-small` |
| TTS URL / model / voice | `Synthesize Reply (Voice)` | `http://speaches:8000/v1/audio/speech` / `speaches-ai/Kokoro-82M-v1.0-ONNX` / `af_heart` |
| History window | `Get recent history1`, `Get Recent History (Voice)` | 10 messages |
| Rate limit | `If4`, `Voice Rate Limit Gate` | < 10 messages per 60 s |
| Request timeout | Ollama / Speaches nodes | 60 s |
| Reminder poll interval | `Cron / Schedule Trigger1` | 1 minute |
| Reminder batch size | `fetch due reminders1` | 50 |
| Timezone | Workflow settings + reminder prompt | `Asia/Karachi` (UTC+5) |
| Admin alert chat | Error handler | `YOUR_TELEGRAM_ADMIN_CHAT_ID` |

To change the user timezone for reminders, edit the system prompt in `HTTP Request3` (it tells the model to subtract 5 hours to convert to UTC) and the workflow timezone setting.

---

## Usage

| Action | What to send |
|--------|--------------|
| Register | `/register` |
| Chat | Any text message |
| Voice chat | Record a voice note; receive text + audio reply |
| Set a reminder | `remind me to call John tomorrow at 5pm` |

Reminders arrive as `⏰ Reminder: call John` at the due time.

---

## Security Notes

- **Never commit credentials.** The exported workflows contain none; keep it that way (n8n exports include credential IDs and instance IDs, so re-sanitize before each commit).
- Restrict who can use the bot: registration is currently open to anyone who finds it. Consider an allow-list of Telegram IDs.
- `password_hash` is stored as the placeholder `n8n-managed`; it is not a real credential. Don't reuse the column for real passwords without proper hashing.
- Keep Ollama and Speaches on a private network; they have no authentication.
- The default model is an *abliterated* (safety-reduced) variant. If the bot is exposed to other users, consider a standard instruct model.
- Reminder text and chat content are stored in plain text in PostgreSQL. Secure and back up the database accordingly.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Bot doesn't respond | Workflow inactive, or n8n isn't reachable over HTTPS (Telegram webhook). Check `WEBHOOK_URL`. |
| "Credential not found" after import | Credentials were stripped; select them in each Telegram/Postgres node. |
| Voice note fails with STT error | Speaches not reachable at `http://speaches:8000`, or the Whisper model wasn't downloaded. |
| No audio reply | TTS model missing in Speaches, or the Kokoro model name is wrong. |
| Reminder time is off | Model mis-converted the timezone; small models are inconsistent. Use a larger model or parse times in code. |
| Reminders stuck in `sending` | n8n crashed between claim and send. Reset: `UPDATE reminders SET status='pending' WHERE status='sending' AND due_at < now() - interval '10 minutes';` |
| "You're sending messages a bit fast" | Rate limit hit (10 msgs/min). Adjust the threshold in the IF nodes. |
| Replies are slow | 3B model on CPU; use a GPU, a smaller quantization, or raise node timeouts. |

---

## Known Limitations & Roadmap

- Reminder detection is keyword-based (`contains "remind"`) and only on the **text** path; voice notes always go to normal chat.
- Voice path logs messages **after** replying, so a logging failure won't block the response but can drop history.
- Reminder parsing depends on a small LLM returning valid JSON; there's no check that `due_at` is in the future.
- No allow-list or admin approval for `/register`.
- Ideas: list/cancel reminders, language auto-detection, voice-triggered reminders, per-user timezones, conversation reset command, streaming responses.

---

## Author

**Amanullah Khan**

- **Role:** Developer & Maintainer
- **Location:** Pakistan
- **GitHub:** [GitHub Profile](https://github.com/amanullahykhan)
- **LinkedIn:** [LinkedIn](https://www.linkedin.com/in/amanullahykhan/)

---

## License

Choose a license for your repository (e.g., MIT) and add a `LICENSE` file.
