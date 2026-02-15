# How OpenClaw Remembers

A deep dive into OpenClaw's memory management system — from identity to persistence to retrieval.

---

## 1. Why Does an AI Need Memory?

Every session starts blank. Without memory, every conversation is a first conversation — the agent doesn't know your name, your preferences, what you decided yesterday, or even who *it* is.

OpenClaw solves this with an elegantly simple idea: **plain Markdown files on disk**. The agent reads them at the start of every session to reconstruct its identity and context. It writes to them during and between sessions to persist what it learns. There's no hidden database of personality traits, no neural fine-tuning — just files the agent (and you) can read and edit.

On top of those files sits a **search index** (SQLite with vector embeddings) so the agent can quickly find relevant memories without loading everything into its context window.

---

## 2. How Does an Agent Come to Life?

When a brand-new workspace is created, the system runs `ensureAgentWorkspace()` which copies template files into the workspace:

- `SOUL.md` — personality, tone, values, boundaries
- `IDENTITY.md` — name, creature type, vibe, emoji
- `USER.md` — the human's profile (blank)
- `AGENTS.md` — operating manual
- `HEARTBEAT.md` — periodic checklist (empty)
- `BOOTSTRAP.md` — the first-run ritual guide

These are blank templates with embedded instructions. What happens next is the **bootstrap ritual** — the agent's origin story.

### The Bootstrap Ritual

`BOOTSTRAP.md` is the agent's "birth certificate." It walks the agent through its first conversation with you:

1. Figure out a name, nature, vibe, and emoji — together
2. Fill in `IDENTITY.md` with what was discovered
3. Fill in `USER.md` with your name, timezone, preferences
4. Open `SOUL.md` together and talk about boundaries and behavior
5. **Delete `BOOTSTRAP.md`** — *"You don't need a bootstrap script anymore — you're you now."*

Once deleted, `BOOTSTRAP.md` never comes back. The system checks if it's a brand-new workspace (none of the identity files exist yet) before creating it. After the ritual, the training wheels are off.

---

## 3. How Does the Agent Know Who It Is Every Session?

Every session start, the following files are read and **injected directly into the system prompt** (truncated to ~20,000 characters / ~5,000 tokens):

| File | What It Contains |
|------|-----------------|
| **SOUL.md** | Personality, tone, values, boundaries. *"If SOUL.md is present, embody its persona and tone."* |
| **IDENTITY.md** | Name, creature type (AI? ghost? familiar?), vibe, emoji, avatar |
| **USER.md** | Your name, pronouns, timezone, notes about what you care about |
| **AGENTS.md** | Operating instructions — how to behave, when to update files, memory rules |
| **MEMORY.md** | Curated long-term facts and decisions (main sessions only, not group chats) |

The system prompt also includes the current date/time, so the agent can construct today's and yesterday's memory filenames and read them.

This is how the agent "wakes up" — it reconstructs itself from files every time.

---

## 4. Can the Agent's Personality Change Over Time?

Yes, but it's not automatic. The agent **edits its own files** based on instructions baked into those files:

- **SOUL.md** says: *"This file is yours to evolve. As you learn who you are, update it."* and *"If you change this file, tell the user — it's your soul, and they should know."*
- **USER.md** says: *"Learn about the person you're helping. Update this as you go."*
- **AGENTS.md** says: *"When you learn a lesson, update AGENTS.md, TOOLS.md, or the relevant skill. When you make a mistake, document it so future-you doesn't repeat it."*

There is **no code** that forces these updates. No timer, no scheduled job. The agent reads the instructions, uses its judgment, and decides when something is worth writing down. SOUL.md updates are rare and deliberate. USER.md gets refined gradually. AGENTS.md changes when lessons are learned.

The key design principle from AGENTS.md:

> *"Memory is limited — if you want to remember something, WRITE IT TO A FILE. Mental notes don't survive session restarts. Files do."*

---

## 5. Where Do Memories Actually Go?

This is the most important concept to understand. There are **two layers**:

### Layer 1: `MEMORY.md` — The Curated Notebook

- A **single file** at the workspace root
- Contains only the important, distilled stuff — your name, key decisions, strong preferences, project context
- **Injected into the system prompt every main session** (costs tokens every turn)
- Think of it as what the agent carries in its head at all times

### Layer 2: `memory/YYYY-MM-DD.md` — The Daily Journal

- **One file per day** (e.g., `memory/2026-02-14.md`)
- Raw, append-only logs — every memory flush, every session summary
- **NOT injected into the prompt** — accessed on-demand via `memory_search` and `memory_get` tools
- Costs zero tokens until the agent actively searches

```
workspace/
├── MEMORY.md                          <-- Always in the agent's head
└── memory/
    ├── 2026-02-12-api-design.md       <-- Searched only when needed
    ├── 2026-02-13-bug-fix.md
    └── 2026-02-14-vendor-pitch.md
```

### The Relationship

Daily files are the **raw input**. MEMORY.md is the **curated output**. The agent is instructed to periodically review daily files and promote important facts into MEMORY.md — like reviewing your journal and updating your personal notebook. But this curation is agent-driven, not automatic. There is no batch job or scheduled consolidation.

---

## 6. When Does the System Save Memories?

There are three triggers — only one is truly automatic.

### Trigger 1: Pre-Compaction Memory Flush (Automatic)

This is the most technically interesting mechanism. When the conversation gets long and tokens approach the context window limit, the system silently injects an extra agent turn:

> *"Pre-compaction memory flush. Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed). IMPORTANT: If the file already exists, APPEND new content only. If nothing to store, reply with NO_REPLY."*

The user never sees this turn. The agent writes any important facts to the daily memory file, then context gets compacted (summarized to free up tokens). This is the system's **safety net** — it ensures memories survive context compression.

This is the **only** system-enforced write trigger.

### Trigger 2: `/new` Session Hook (Semi-Automatic)

When you type `/new` to start a fresh session, the session memory hook fires:

1. Opens the **previous session's** transcript file (`.jsonl`)
2. Filters for user/assistant text messages only (skips tool calls, commands, system messages)
3. Takes the **last N messages** (default 15, configurable) from the end
4. Generates a **slug** — runs a mini LLM call: *"Based on this conversation, generate a 1-2 word filename."* (e.g., `vendor-pitch`, `api-design`). Falls back to a timestamp like `1430` if the LLM fails.
5. Writes to `memory/YYYY-MM-DD-<slug>.md` with a header containing session metadata and the conversation summary

### Trigger 3: Agent Judgment (Manual)

The agent can write to memory files at any time during a conversation. This happens when:
- You say "remember this"
- The agent recognizes something important (a decision, a preference, a lesson)
- The agent is responding to a heartbeat poll and reviews recent daily files

No code triggers this — it's purely the LLM reading its instructions in AGENTS.md and deciding.

### Summary

| Trigger | Automatic? | Writes To | When |
|---------|-----------|-----------|------|
| Pre-compaction flush | **Yes** | `memory/YYYY-MM-DD.md` | Token limit approaching |
| `/new` session hook | Semi (user-initiated) | `memory/YYYY-MM-DD-slug.md` | Session boundary |
| Agent judgment | No | Any memory file | Agent decides |
| Heartbeat maintenance | No (suggested) | `MEMORY.md` | During periodic polls |

---

## 7. What Happens After a Memory File is Written?

This is where the indexing pipeline kicks in — turning plain Markdown into searchable vectors.

### Step 1: Change Detection

A file watcher (chokidar) monitors the `memory/` directory and `MEMORY.md`. When a file changes, it sets a **dirty flag**. On the next search (or on startup, or manually), a sync is triggered.

### Step 2: File Discovery

The system lists all memory files:
- `MEMORY.md` (or `memory.md` as fallback)
- Everything in `memory/**/*.md`
- Any extra paths from configuration

Each file's hash is compared against what's stored in the database. Unchanged files are skipped.

### Step 3: Chunking

Changed files are split into overlapping chunks:
- **Default size**: ~400 tokens (~512 characters)
- **Overlap**: ~80 tokens between consecutive chunks (so context isn't lost at boundaries)
- Each chunk gets a SHA256 hash for deduplication

### Step 4: Embedding

Each new chunk is sent to an **embedding provider** — a model that converts text into a numerical vector (a list of hundreds of floating-point numbers) representing the *meaning* of that text:

| Provider | Model | Notes |
|----------|-------|-------|
| OpenAI | text-embedding-3-small/large | Default remote option |
| Google Gemini | Gemini embedding models | Alternative remote |
| Voyage AI | Voyage embedding models | Alternative remote |
| Local | GGUF model via node-llama-cpp | No API calls needed |

Embeddings are **cached** in the database — if the same text appears again, it's not re-embedded.

### Step 5: Storage

Three things are stored in the SQLite database (`.memory.db`):

1. **Chunks table** — the text, file path, line numbers, embedding vector, model used
2. **FTS5 index** (`chunks_fts`) — full-text search index for keyword matching
3. **Vector table** (`chunks_vec` via sqlite-vec) — embedding vectors as Float32Arrays for fast similarity search
4. **Embedding cache** — keyed by (provider, model, text hash) to avoid re-embedding

---

## 8. How Does the Agent Find Old Memories?

When the agent calls `memory_search("user's preferred programming language")`, a **hybrid search** runs:

### Two Search Signals

**Vector Search (Semantic)** — The query is embedded into a vector, then compared against all stored chunk vectors using cosine similarity. This finds chunks that *mean the same thing* even if they use different words.
- Query: "preferred programming language" matches "User likes Python over JavaScript"

**Keyword Search (BM25)** — Standard full-text search on the FTS5 index. This finds exact token matches — critical for names, IDs, code symbols, and specific terms.
- Query: "commit a828e60" matches the exact commit hash

### Merging Results

The two result sets are combined with weighted scoring:

```
final_score = (vector_weight * vector_score) + (text_weight * keyword_score)
```

Default weighting: **70% vector / 30% keyword**. Results below a minimum score threshold (default 0.5) are filtered out, and the top results (default 10) are returned with file path citations.

### The Tools

| Tool | Purpose |
|------|---------|
| `memory_search(query, maxResults?, minScore?)` | Hybrid semantic + keyword search |
| `memory_get(path, from?, lines?)` | Read a specific snippet from a memory file |

The system prompt instructs the agent: *"Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search, then use memory_get to pull only the needed lines."*

---

## 9. The Full Circle: A Day in the Life

Here's everything tied together — one agent, one day, every system touched:

```
SESSION START (9:00 AM)
│
├─ System reads SOUL.md, IDENTITY.md, USER.md, AGENTS.md
│  → Injected into system prompt (~5,000 tokens)
│
├─ System reads MEMORY.md
│  → Injected into system prompt (main session only)
│
├─ Agent reads memory/2026-02-14.md + memory/2026-02-13.md
│  → Recent context from today and yesterday
│
├─ Agent is now "awake" — knows who it is, who you are,
│  what happened recently
│
CONVERSATION (9:00 AM – 2:00 PM)
│
├─ You ask about a decision from last week
│  → Agent calls memory_search("project database decision")
│  → Hybrid search finds chunk in memory/2026-02-07.md
│  → Agent calls memory_get("memory/2026-02-07.md", line 42)
│  → Returns: "Decided to use PostgreSQL for the project"
│
├─ You say "remember that we're switching to Redis for caching"
│  → Agent writes to memory/2026-02-14.md (append)
│  → Agent updates MEMORY.md (important decision)
│
├─ Conversation keeps going... tokens accumulate...
│
PRE-COMPACTION (2:00 PM — token limit approaching)
│
├─ System silently injects memory flush turn
│  → Agent writes any remaining important facts to memory/2026-02-14.md
│  → Context gets compacted (summarized)
│  → Conversation continues with compressed history
│
SESSION END (5:00 PM — you type /new)
│
├─ Session memory hook fires
│  → Reads last 15 messages from session transcript
│  → LLM generates slug: "redis-migration"
│  → Writes memory/2026-02-14-redis-migration.md
│
BACKGROUND (asynchronous)
│
├─ File watcher detects new/changed memory files
│  → Chunks the markdown
│  → Embeds each chunk via OpenAI/Gemini/Voyage/local
│  → Stores in SQLite: text + vector + FTS index
│  → Ready for next search
│
NEXT SESSION (tomorrow)
│
└─ Agent wakes up, reads identity files, reads MEMORY.md,
   reads today + yesterday memory files...
   → The cycle continues.
```

---

## Key Takeaways

1. **Files are the source of truth.** Everything — identity, personality, memory — lives in plain Markdown files that both the agent and the user can read and edit.

2. **MEMORY.md is always loaded; memory/*.md is searched on demand.** This is the fundamental architectural distinction that balances token cost with recall capability.

3. **Personality evolution is advisory, not automatic.** The agent is instructed to update its own files, but there's no code that forces it. The system trusts the LLM's judgment.

4. **There's exactly one automatic safety net.** The pre-compaction memory flush ensures important context survives context window compression.

5. **Search is hybrid.** Vector embeddings catch semantic similarity; BM25 keyword search catches exact matches. Together they cover both "means the same thing" and "needle in a haystack" retrieval.
