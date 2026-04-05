# BEAST Pipeline — Full System Spec

## 1. Topic Sourcing (TopicMaster)

5 sources feed topics into `topic_brain.db` (SQLite):

| Source | Priority | Method |
|--------|----------|--------|
| Question Bank (1,124 Qs) | 0.5 | One-time bulk import |
| Pillar Research (7 docs) | 0.6 | One-time bulk import |
| Mentor Scraper (Google Sheet) | 0.8 | Sheets API, live read |
| Reddit Brand Bot (VPS SQLite) | 0.9 | Direct DB read from `mined_insights` |
| Intl Creator Frameworks (100) | 0.7 | Parse + Haiku generates variants |

**Harvester** runs daily 6AM IST — pulls new topics, batch-classifies via Haiku (pillar, awareness, perspective, business_type), inserts into `topics` table with status `queued`.

---

## 2. Organization — Pillars + Awareness

### 7 Content Pillars

| Pillar | Weight (Launch) | Core Transformation |
|--------|----------------|---------------------|
| MONEY | 20% | "Cash is coming in" -> "I know my real profit" |
| ACQUIRE | 25% | "I'll open and they'll come" -> "I have a system" |
| EXPERIENCE | 15% | "I sell products" -> "I design experiences" |
| RETAIN | 10% | "I need new customers" -> "Same ones more often" |
| PRICING | 10% | "I match their price" -> "I make my price feel worth it" |
| OPERATIONS | 10% | "I run the shop" -> "The shop runs on systems" |
| PEOPLE | 10% | "I can't trust anyone" -> "I build people who build this" |

Weights are phase-based (launch / authority / community) — stored in `pillar_weights` table.

### 3 Awareness Levels

- **INVISIBLE** — owner doesn't know the problem exists
- **FRUSTRATION** — feels it daily, can't name it
- **STUCK** — knows the problem, can't move forward

### Clustering

Haiku text comparison (no embeddings), batches of 30 topics, groups by root problem into `clusters` table.

---

## 3. Scoring — 5-Signal Composite Algorithm

| Signal | Weight | What it measures |
|--------|--------|-----------------|
| Pillar Deficit | 30% | Gap between target % and actual published % (last 30 days) |
| Source Priority | 25% | Direct from `sources.priority` (0.5-0.9) |
| Freshness | 20% | Decay 0.03/day from 1.0; evergreen sources = 0.7 flat |
| Awareness Balance | 15% | How underrepresented this awareness level is |
| Perspective Freshness | 10% | Has this (cluster, perspective) combo been produced before? |

**Formula:**

```
composite_score = (pillar_deficit × 0.30)
               + (source_priority × 0.25)
               + (freshness × 0.20)
               + (awareness_balance × 0.15)
               + (perspective_freshness × 0.10)
               + manual_priority
```

Manual priority: +0.5 (boost) or -0.3 (deprioritize).

**Picker** applies diversity constraints: max 2 same pillar, max 2 same cluster, min 2 awareness stages per batch.

---

## 4. Content Pipeline — 7 Agents in Sequence

### Phase 1: Strategy Gate (Topic -> Scripts)

#### Step 1 — NADAI (Style DNA Selector)

- Input: topic's pillar + awareness level
- Looks up `style-config.json` (pillar x awareness -> tags like `strong-hooks`, `emotional`, `practical`)
- Scores creator profiles by tag overlap, picks top 2-3
- Layers them on creator's baseline voice (never overrides baseline)
- Output: ~300-word STYLE DNA text block

#### Step 2 — THRIVA (Script Architect)

- Input: topic + Style DNA
- Generates 3 variations in 3 mandatory formats:
  - V1: Counter-Intuitive (flip a belief)
  - V2: Story-First (vivid anecdote -> insight)
  - V3: Data-Shock (surprising stat -> explain)
- Each variation has Part A (narrative) + Part B (shooting script with markers)
- Shooting script markers: `[TEXT HOOK]`, `[VISUAL]`, `[SPEAK]`, `[B-ROLL]`, `[CUT BACK]`, `[ACTION STEPS]`, `[CTA]`
- Hard rules: 1 relatable analogy, 1 personal authority line, energy spike in first `[SPEAK]`, active CTA with question, 3 solution steps

#### Step 3 — KELLA (Direct Response Strategist)

- Input: all 3 Thriva variations + Style DNA
- Per variation, produces sections A-F:
  - A: Quick diagnosis (5-7 bullets on what's weak)
  - B: Rewritten script (spoken language, max 100 words)
  - C: Why better (3-5 bullets)
  - **D: Teleprompter script** (70-84 words, energy markers)
  - E: Visual hook (first 3 seconds)
  - F: Music & mood direction
- Energy markers: `[PUNCH]`, `[STEADY]`, `[DROP]`, `[RISE]`
- Teleprompter rules: conversational tone (not telegraphic), CAPS only for emphasis, preserve natural fillers

#### Step 4 — SENSEI (Variation Evaluator)

- Scores all 3 on 5 criteria:
  - Audience Resonance (30%)
  - Hook Strength (25%)
  - Story Clarity (20%)
  - Action Quality (15%)
  - Performance Signal (10%)
- Ranks 1-2-3 with rationale
- **Does NOT pick** — sends rankings to user, user picks winner via Telegram or dashboard

### Phase 2: Packaging (Auto after user picks)

#### Step 5 — TUBEREEL (Social Media Analyst)

- Input: chosen variation's teleprompter + text hook
- Output:
  - Instagram caption (8-12 lines, 5 hashtags)
  - Pinned first comment (question trigger)
  - 3 YouTube Shorts title options (under 60 chars)
  - YouTube description (full English for SEO, 8-12 lines)
  - 8-10 SEO keywords

#### Step 6 — BEROLLER (Visual Director)

- Input: chosen teleprompter script
- Output: 9-10 B-roll scene prompts (3-6 seconds each, 9:16 vertical)
- Scene 1 is the HOOK (scroll-stopper, emotional punch)
- Per scene: exact script line covered, video prompt, camera type, location context, style, duration
- Aesthetic: phone-recorded look, clean businesses, local context

#### Step 7 — THUMBSTER (Thumbnail Director)

- Generates 5-6 text hook variations (3-6 words)
- User picks one, then generates ONE thumbnail blueprint (best CTR lever)
- Design rules: max 3 elements (face + text hook + one graphic), local background, specific color palette
- Outputs a prompt for AI image generation

### Phase 3: Distribution (Content Waterfall)

**DISTRIBUTOR** converts the script into 5 platform versions — all as drafts, never auto-publish:

| Platform | Length | Style |
|----------|--------|-------|
| Blog | 600-1000 words | Personal story-driven, opens with anecdote |
| Medium | 800-1200 words | Different angle than blog, deeper exploration |
| LinkedIn | 150-200 words | Hook line 1, one insight, question CTA |
| Twitter/X | 260 chars max | Single tweet, sharpest insight |
| Threads | 200-300 chars | 3-5 lines, pure text, no hashtags |

Voice rules across all: humble tone, skip apostrophes, no AI-sounding words (delve, crucial, pivotal, game-changer), ".." for pauses.

---

## 5. Database Schema

### `sources` table

```sql
CREATE TABLE sources (
    id          INTEGER PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    priority    REAL NOT NULL DEFAULT 0.5
);
```

### `pillars` table

```sql
CREATE TABLE pillars (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    target_weight   REAL NOT NULL
);
```

### `pillar_weights` table

```sql
CREATE TABLE pillar_weights (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    phase       TEXT NOT NULL,          -- launch, authority, community
    pillar_id   INTEGER REFERENCES pillars(id),
    weight      REAL NOT NULL,
    active      BOOLEAN DEFAULT 0
);
```

### `topics` table (core)

```sql
CREATE TABLE topics (
    id                          INTEGER PRIMARY KEY AUTOINCREMENT,
    raw_text                    TEXT NOT NULL,
    normalized_text             TEXT,
    source_id                   INTEGER REFERENCES sources(id),
    pillar_id                   INTEGER REFERENCES pillars(id),
    awareness                   TEXT,       -- unaware, problem_aware, solution_aware
    perspective                 TEXT,       -- owner_voice, consumer_voice, framework_voice, mentee_voice, trend_voice
    business_type               TEXT,       -- restaurant, retail, salon, universal
    cluster_id                  INTEGER REFERENCES clusters(id),

    -- Scoring
    pillar_deficit_score        REAL DEFAULT 0.0,
    source_score                REAL DEFAULT 0.0,
    freshness_score             REAL DEFAULT 0.0,
    awareness_balance_score     REAL DEFAULT 0.0,
    perspective_score           REAL DEFAULT 0.0,
    composite_score             REAL DEFAULT 0.0,

    -- Lifecycle
    status                      TEXT DEFAULT 'queued',  -- queued -> scored -> marked -> in_beast -> published -> archived

    -- Metadata
    creator_name                TEXT,
    framework_name              TEXT,
    tamil_hook                  TEXT,
    source_url                  TEXT,
    mentee_id                   TEXT,

    -- Timestamps
    ingested_at                 DATETIME DEFAULT CURRENT_TIMESTAMP,
    scored_at                   DATETIME,
    sent_to_beast_at            DATETIME,
    published_at                DATETIME,

    -- Manual override
    manual_priority             INTEGER DEFAULT 0,
    notes                       TEXT
);

CREATE INDEX idx_topics_status ON topics(status);
CREATE INDEX idx_topics_pillar ON topics(pillar_id);
CREATE INDEX idx_topics_composite ON topics(composite_score DESC);
CREATE INDEX idx_topics_cluster ON topics(cluster_id);
```

### `clusters` table

```sql
CREATE TABLE clusters (
    id                      INTEGER PRIMARY KEY AUTOINCREMENT,
    root_problem            TEXT NOT NULL,
    pillar_id               INTEGER REFERENCES pillars(id),
    topic_count             INTEGER DEFAULT 0,
    perspectives_covered    TEXT DEFAULT '[]',   -- JSON array
    content_produced        TEXT DEFAULT '[]',   -- JSON array
    created_at              DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### `publish_log` table

```sql
CREATE TABLE publish_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    topic_id        INTEGER REFERENCES topics(id),
    pillar_id       INTEGER REFERENCES pillars(id),
    awareness       TEXT,
    perspective     TEXT,
    published_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_publish_pillar ON publish_log(pillar_id);
CREATE INDEX idx_publish_date ON publish_log(published_at);
```

### `creator_frameworks` table

```sql
CREATE TABLE creator_frameworks (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    creator_name    TEXT NOT NULL,
    framework_name  TEXT NOT NULL,
    pillar_id       INTEGER REFERENCES pillars(id),
    core_concept    TEXT NOT NULL,
    tamil_hook      TEXT,
    used_count      INTEGER DEFAULT 0,
    last_used_at    DATETIME
);
```

### Pipeline state tables

```sql
CREATE TABLE pipeline_runs (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    topic_id        INTEGER REFERENCES topics(id),
    run_id          TEXT UNIQUE,
    status          TEXT,           -- running, paused, completed, failed
    current_step    TEXT,
    started_at      DATETIME,
    completed_at    DATETIME
);

CREATE TABLE pipeline_steps (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id          TEXT REFERENCES pipeline_runs(run_id),
    step            TEXT,           -- nadai, thriva, kella, sensei, tubereel, beroller, thumbster
    step_order      INTEGER,
    status          TEXT,           -- pending, running, completed, failed, skipped
    started_at      DATETIME,
    completed_at    DATETIME,
    output          TEXT,           -- JSON
    metadata        TEXT            -- JSON
);

CREATE TABLE pipeline_outputs (
    run_id                  TEXT REFERENCES pipeline_runs(run_id),
    nadai_blend             TEXT,       -- JSON
    thriva_v1               TEXT,       -- JSON
    thriva_v2               TEXT,
    thriva_v3               TEXT,
    kella_v1                TEXT,       -- JSON
    kella_v2                TEXT,
    kella_v3                TEXT,
    sensei_scores           TEXT,       -- JSON
    sensei_pick             TEXT,
    teleprompter            TEXT,
    teleprompter_edited     INTEGER DEFAULT 0,
    tubereel_output         TEXT,       -- JSON
    beroller_prompts        TEXT,       -- JSON
    thumbster_output        TEXT,       -- JSON
    shirt_color             TEXT,
    posted_date             DATE,
    youtube_url             TEXT,
    instagram_url           TEXT
);
```

---

## 6. API Endpoints

Base URL: `http://<VPS_IP>:<PORT>/api/topicmaster`

```
GET  /queue?pillar=&status=&limit=20    — Ranked topic queue
GET  /topic/:id                         — Topic details
GET  /balance                           — Pillar coverage stats
GET  /stats                             — Queue stats
POST /pick              { count: N }    — Pick top N topics with diversity constraints
POST /harvest                           — Manual harvest trigger
POST /phase             { phase: '...'} — Switch pillar weight phase

POST /topic/:id/start-pipeline          — Returns { runId }
POST /pipeline/:runId/step              — Update step status
POST /pipeline/:runId/output            — Save agent outputs
POST /pipeline/:runId/pick  { variation: "v1" }  — User picks variation
POST /pipeline/:runId/complete          — Mark pipeline done

GET  /pipeline                          — List all pipeline runs
```

---

## 7. Infrastructure

| Component | Detail |
|-----------|--------|
| VPS | Ubuntu, Node.js, PM2 managed |
| Database | SQLite (topic_brain.db, queue.db) |
| AI (content) | Claude CLI (`claude -p`) — zero API cost |
| AI (classify) | Claude Haiku for classification, clustering |
| Dashboard | Express.js web app |
| Vault | Obsidian, git-synced to VPS |
| Notifications | Telegram bot |
| Scripts saved as | `beast-scripts/row-{id}-{slug}.md` in vault |
| Batch mode | Processes topics one by one, 5-min pick timeout then skip |
| Beast-runner | PM2 process, watches every 30s |

### Cron Schedule

| Time (IST) | Job |
|------------|-----|
| 6:00 AM | TopicMaster Harvester |
| 6:30 AM | AI Digest |
| 8:00 AM | Scout morning crawl |
| 8:00 PM | Scout evening crawl |
| Every 30 min | Batch scheduler checks for pending runs |
| Every 30 min | Distributor publishes pending drafts |

---

## 8. Style DNA System (Nadai)

### style-config.json structure

Maps pillar x awareness -> style tags:

```json
{
  "ACQUIRE": {
    "INVISIBLE": ["strong-hooks", "shock-hooks", "data-heavy", "high-energy"],
    "FRUSTRATION": ["emotional", "storytelling", "question-hooks", "conversational"],
    "STUCK": ["practical", "calm-authority", "educational", "strong-hooks"]
  },
  "MONEY": {
    "INVISIBLE": ["strong-hooks", "shock-hooks", "data-heavy", "high-energy"],
    "FRUSTRATION": ["emotional", "question-hooks", "conversational", "storytelling"],
    "STUCK": ["practical", "calm-authority", "educational", "analytical"]
  }
}
```

### Creator profiles

Each profile JSON contains: `auto_tags`, `hooks`, `energy`, `signature_phrases`, `viral_patterns`.

Nadai process:
1. Look up required tags from style-config.json
2. Score each creator profile by tag overlap (min 2 tags)
3. Select top 2-3 matching creators
4. Load baseline voice as foundation
5. Generate STYLE DNA text block (< 300 words)
6. Pass to Thriva, Kella, Sensei as context

---

## 9. Pipeline Flow Diagram

```
[TopicMaster]
    |
    |-- Harvest (5 sources, daily 6AM)
    |-- Classify (Haiku batch)
    |-- Cluster (Haiku text comparison)
    |-- Score (5-signal composite)
    |-- Pick (diversity-constrained)
    |
    v
[BEAST Phase 1: Strategy]
    |
    |-- Nadai   -> Style DNA
    |-- Thriva  -> 3 script variations (counter-intuitive, story, data-shock)
    |-- Kella   -> Polish all 3 (sections A-F, teleprompter scripts)
    |-- Sensei  -> Rank 1-2-3
    |
    |  ** USER PICKS WINNER **
    |
    v
[BEAST Phase 2: Packaging]
    |
    |-- TubeReel   -> Captions, titles, descriptions, SEO
    |-- BEroller   -> 9-10 B-roll scene prompts
    |-- Thumbster  -> Thumbnail blueprint + AI image prompt
    |
    v
[BEAST Phase 3: Distribution]
    |
    |-- Blog (600-1000 words)
    |-- Medium (800-1200 words)
    |-- LinkedIn (150-200 words)
    |-- Twitter/X (260 chars)
    |-- Threads (200-300 chars)
    |
    |  All as DRAFTS — never auto-publish
    |
    v
[Publish Log -> Re-scores TopicMaster]
```

---

## 10. Environment Variables

```env
# VPS
VPS_HOST=<your-vps-ip>
VPS_PORT=9090

# Telegram
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_CHAT_ID=<your-chat-id>

# Google Sheets (for Mentor Scraper source)
GOOGLE_SHEETS_ID=<your-sheet-id>
GOOGLE_SHEETS_CREDENTIALS=<path-to-credentials.json>

# Dashboard Auth
DASHBOARD_USERNAME=<your-username>
DASHBOARD_PASSWORD=<your-password>

# Vault
VAULT_PATH=<path-to-obsidian-vault>
VAULT_GIT_REMOTE=<git-remote-url>
```

---

## 11. Key Rules & Constraints

### Pipeline Rules
- NEVER skip a step — every topic through all phases
- NEVER fabricate data — report errors honestly
- NEVER auto-publish — human approval required at pick step and publish step
- If Sensei fails/times out: default to V1 with note
- Scripts saved to vault BEFORE Telegram notify
- TopicMaster DB is single source of truth

### Script Quality Rules
- Max 100 words (aim 70-80) for teleprompter
- Min 3 action steps (practical, doable TODAY)
- Min 1 relatable analogy per variation
- Min 1 personal authority line
- Min 1 active CTA with question (not passive)
- Min 2 energy modulation markers in teleprompter
- Triple alignment: visual hook + text hook + spoken hook must match

### Voice Quality Rules
- Conversational, not telegraphic
- CAPS for emphasis ONLY (not every other word)
- Preserve natural fillers (intentional voice markers)
- No jargon, no hype, no coach-speak
- One idea per line
- Respectful forms only
