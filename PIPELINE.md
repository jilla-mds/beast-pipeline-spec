# Multi-Agent Content Pipeline — Full System Spec

## Overview

This system automates the full lifecycle of short-form video content: **source topics -> score for virality -> generate scripts -> package assets -> distribute across platforms**. It uses 7 specialized AI agents orchestrated sequentially, with human approval gates at key decision points.

---

## 1. Topic Sourcing (TopicMaster)

Multiple sources feed topics into a central SQLite database. Each source has a priority weight that influences scoring.

### Source Types

| Source Type | Priority | Method |
|-------------|----------|--------|
| Question Bank (bulk) | 0.5 | One-time import |
| Niche Research Docs | 0.6 | One-time import |
| CRM / Mentorship Notes | 0.8 | Sheets API, live read |
| Social Listening (Reddit, etc.) | 0.9 | Direct DB read from scout bot |
| Creator Framework Library | 0.7 | Parse + Haiku generates variants |

**Harvester** runs daily — pulls new topics, batch-classifies via Haiku (pillar, awareness, perspective, business_type), inserts into `topics` table with status `queued`.

---

## 2. Organization — Pillars + Awareness

### Content Pillars

Define your own pillars — categories that cover your niche. Each pillar has a target weight (% of total content). Weights shift across growth phases.

Example structure:

| Pillar | Weight | Description |
|--------|--------|-------------|
| PILLAR_A | 25% | Core topic area |
| PILLAR_B | 20% | Second priority |
| PILLAR_C | 15% | Supporting area |
| ... | ... | ... |

Weights are **phase-based** — you define 2-3 phases (e.g., launch, growth, community) with different weight distributions stored in `pillar_weights` table. Only one phase is active at a time.

### 3 Awareness Levels

Topics address audiences at different problem-awareness stages:

- **UNAWARE** — audience doesn't know the problem exists. Hook: surprising revelation, hidden mistake.
- **PROBLEM_AWARE** — feels the pain daily but can't name the root cause. Hook: root cause exposure, validation.
- **SOLUTION_AWARE** — knows the problem, stuck on execution. Hook: unlock one specific actionable thing.

### Clustering

Haiku text comparison (no embeddings) groups topics by root problem in batches of 30. This prevents producing multiple videos about the same underlying issue.

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

#### Agent 1 — STYLE SELECTOR

- Input: topic's pillar + awareness level
- Looks up a config mapping (pillar x awareness -> style tags like `strong-hooks`, `emotional`, `practical`)
- Scores reference creator profiles by tag overlap, picks top 2-3
- Layers them on your baseline voice profile (never overrides baseline)
- Output: ~300-word STYLE DNA text block passed to all downstream agents

#### Agent 2 — SCRIPT ARCHITECT

- Input: topic + Style DNA
- Generates **3 variations** in 3 mandatory formats:
  - V1: **Counter-Intuitive** (flip a belief)
  - V2: **Story-First** (vivid anecdote -> insight)
  - V3: **Data-Shock** (surprising stat -> explain)
- Each variation has:
  - Part A: Narrative summary (why this angle works)
  - Part B: Shooting script with markers: `[TEXT HOOK]`, `[VISUAL]`, `[SPEAK]`, `[B-ROLL]`, `[CUT BACK]`, `[ACTION STEPS]`, `[CTA]`
- Hard rules: 1 relatable analogy, 1 personal authority line, energy spike in first `[SPEAK]`, active CTA with question, 3 actionable solution steps

#### Agent 3 — SCRIPT OPTIMIZER

- Input: all 3 variations + Style DNA
- Per variation, produces sections A-F:
  - **A:** Quick diagnosis (5-7 bullets on what's weak)
  - **B:** Rewritten script (spoken language, max 100 words)
  - **C:** Why better (3-5 bullets)
  - **D: Teleprompter script** (70-84 words with energy markers)
  - **E:** Visual hook (first 3 seconds design)
  - **F:** Music & mood direction
- Energy markers: `[PUNCH]` (high energy), `[STEADY]` (default), `[DROP]` (soft/intimate), `[RISE]` (building)
- Teleprompter rules: conversational tone (not telegraphic), CAPS only for emphasis, preserve natural fillers, one idea per line

#### Agent 4 — VARIATION EVALUATOR

- Scores all 3 variations on 5 weighted criteria:
  - Audience Resonance (30%)
  - Hook Strength (25%)
  - Story Clarity (20%)
  - Action Quality (15%)
  - Performance Signal (10%)
- Ranks 1-2-3 with rationale
- **Does NOT pick** — sends rankings to user. User picks winner via Telegram or dashboard.

### Phase 2: Packaging (Auto-runs after user picks)

#### Agent 5 — SOCIAL MEDIA ANALYST

- Input: chosen variation's teleprompter + text hook
- Output:
  - Instagram caption (8-12 lines, hashtags)
  - Pinned first comment (question trigger for engagement)
  - 3 YouTube Shorts title options (under 60 chars)
  - YouTube description (full English for SEO, 8-12 lines)
  - 8-10 SEO keywords

#### Agent 6 — VISUAL DIRECTOR

- Input: chosen teleprompter script
- Output: 9-10 B-roll scene prompts (3-6 seconds each, vertical 9:16)
- Scene 1 is the HOOK (scroll-stopper, emotional punch)
- Per scene: exact script line covered, video prompt, camera type (static/handheld/POV/pan), location context, style notes, duration
- Aesthetic: phone-recorded look, clean environments, local context

#### Agent 7 — THUMBNAIL DIRECTOR

- Generates 5-6 text hook variations (3-6 words each)
- User picks one, then generates ONE thumbnail blueprint (best CTR lever)
- Design rules: max 3 elements (face + text hook + one graphic element), real-world background, specific color palette
- Outputs a single-line prompt for AI image generation tools

### Phase 3: Distribution (Content Waterfall)

A **DISTRIBUTOR** agent converts the original script into platform-specific versions — all saved as drafts, never auto-published:

| Platform | Length | Style |
|----------|--------|-------|
| Blog | 600-1000 words | Personal story-driven, opens with anecdote |
| Medium | 800-1200 words | Different angle than blog, deeper exploration |
| LinkedIn | 150-200 words | Hook in line 1, one insight, question CTA |
| Twitter/X | 260 chars max | Single tweet, sharpest insight |
| Threads | 200-300 chars | 3-5 lines, pure text, no hashtags |

Each platform has its own voice rules and constraints. A cron job checks for scheduled drafts and publishes them at the set time.

---

## 5. Database Schema

### `sources`

```sql
CREATE TABLE sources (
    id          INTEGER PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    priority    REAL NOT NULL DEFAULT 0.5
);
```

### `pillars`

```sql
CREATE TABLE pillars (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    target_weight   REAL NOT NULL
);
```

### `pillar_weights`

```sql
CREATE TABLE pillar_weights (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    phase       TEXT NOT NULL,          -- e.g. launch, growth, community
    pillar_id   INTEGER REFERENCES pillars(id),
    weight      REAL NOT NULL,
    active      BOOLEAN DEFAULT 0
);
```

### `topics` (core)

```sql
CREATE TABLE topics (
    id                          INTEGER PRIMARY KEY AUTOINCREMENT,
    raw_text                    TEXT NOT NULL,
    normalized_text             TEXT,
    source_id                   INTEGER REFERENCES sources(id),
    pillar_id                   INTEGER REFERENCES pillars(id),
    awareness                   TEXT,       -- unaware, problem_aware, solution_aware
    perspective                 TEXT,       -- e.g. owner_voice, consumer_voice, framework_voice
    business_type               TEXT,       -- your niche segments
    cluster_id                  INTEGER REFERENCES clusters(id),

    -- Scoring
    pillar_deficit_score        REAL DEFAULT 0.0,
    source_score                REAL DEFAULT 0.0,
    freshness_score             REAL DEFAULT 0.0,
    awareness_balance_score     REAL DEFAULT 0.0,
    perspective_score           REAL DEFAULT 0.0,
    composite_score             REAL DEFAULT 0.0,

    -- Lifecycle
    status                      TEXT DEFAULT 'queued',
    -- queued -> scored -> marked -> in_pipeline -> published -> archived

    -- Metadata
    creator_name                TEXT,
    framework_name              TEXT,
    hook_text                   TEXT,
    source_url                  TEXT,

    -- Timestamps
    ingested_at                 DATETIME DEFAULT CURRENT_TIMESTAMP,
    scored_at                   DATETIME,
    sent_to_pipeline_at         DATETIME,
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

### `clusters`

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

### `publish_log`

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

### `creator_frameworks`

```sql
CREATE TABLE creator_frameworks (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    creator_name    TEXT NOT NULL,
    framework_name  TEXT NOT NULL,
    pillar_id       INTEGER REFERENCES pillars(id),
    core_concept    TEXT NOT NULL,
    hook_text       TEXT,
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
    step            TEXT,           -- style_selector, script_architect, script_optimizer, evaluator, social_analyst, visual_director, thumbnail_director
    step_order      INTEGER,
    status          TEXT,           -- pending, running, completed, failed, skipped
    started_at      DATETIME,
    completed_at    DATETIME,
    output          TEXT,           -- JSON
    metadata        TEXT            -- JSON
);

CREATE TABLE pipeline_outputs (
    run_id                  TEXT REFERENCES pipeline_runs(run_id),
    style_dna               TEXT,       -- JSON
    script_v1               TEXT,       -- JSON
    script_v2               TEXT,
    script_v3               TEXT,
    optimized_v1            TEXT,       -- JSON
    optimized_v2            TEXT,
    optimized_v3            TEXT,
    eval_scores             TEXT,       -- JSON
    user_pick               TEXT,
    teleprompter            TEXT,
    teleprompter_edited     INTEGER DEFAULT 0,
    social_output           TEXT,       -- JSON
    broll_prompts           TEXT,       -- JSON
    thumbnail_output        TEXT,       -- JSON
    posted_date             DATE,
    video_urls              TEXT        -- JSON
);
```

---

## 6. API Endpoints

Base URL: `http://<VPS_IP>:<PORT>/api/topics`

```
GET  /queue?pillar=&status=&limit=20    — Ranked topic queue
GET  /topic/:id                         — Topic details
GET  /balance                           — Pillar coverage stats
GET  /stats                             — Queue stats
POST /pick              { count: N }    — Pick top N with diversity constraints
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
| AI (content) | Claude CLI (`claude -p`) — zero cost on Max plan |
| AI (classify) | Claude Haiku for classification + clustering |
| Dashboard | Express.js web app |
| Vault | Obsidian, git-synced to VPS |
| Notifications | Telegram bot |
| Scripts saved as | Markdown files in vault |
| Batch mode | Processes topics one by one, configurable pick timeout |
| Runner | PM2 process, polls every 30s |

### Cron Schedule (customize to your timezone)

| Time | Job |
|------|-----|
| 6:00 AM | Harvester (pull from all sources) |
| 7:00 AM | Daily digest / notifications |
| Every 30 min | Batch scheduler checks for pending runs |
| Every 30 min | Distributor publishes scheduled drafts |

---

## 8. Style DNA System

### Config Structure

A JSON config maps pillar x awareness -> style tags:

```json
{
  "PILLAR_A": {
    "UNAWARE": ["strong-hooks", "shock-hooks", "data-heavy", "high-energy"],
    "PROBLEM_AWARE": ["emotional", "storytelling", "question-hooks", "conversational"],
    "SOLUTION_AWARE": ["practical", "calm-authority", "educational", "analytical"]
  }
}
```

### Creator Profiles

Each profile is a JSON file containing: `auto_tags`, `hooks`, `energy`, `signature_phrases`, `viral_patterns`.

**Style Selector process:**
1. Look up required tags from config
2. Score each creator profile by tag overlap (min 2 tags to qualify)
3. Select top 2-3 matching creators
4. Load your baseline voice as foundation
5. Generate STYLE DNA text block (< 300 words)
6. Pass to all downstream agents as context

---

## 9. Pipeline Flow Diagram

```
[TopicMaster]
    |
    |-- Harvest (multiple sources, daily)
    |-- Classify (Haiku batch — pillar, awareness, perspective)
    |-- Cluster (Haiku text comparison — group by root problem)
    |-- Score (5-signal composite)
    |-- Pick (diversity-constrained selection)
    |
    v
[Phase 1: Strategy]
    |
    |-- Style Selector    -> Style DNA
    |-- Script Architect  -> 3 variations (counter-intuitive, story, data-shock)
    |-- Script Optimizer  -> Polish all 3 (sections A-F, teleprompter scripts)
    |-- Evaluator         -> Rank 1-2-3
    |
    |  ** HUMAN APPROVAL GATE — user picks winner **
    |
    v
[Phase 2: Packaging]
    |
    |-- Social Analyst     -> Captions, titles, descriptions, SEO
    |-- Visual Director    -> 9-10 B-roll scene prompts
    |-- Thumbnail Director -> Thumbnail blueprint + AI image prompt
    |
    v
[Phase 3: Distribution]
    |
    |-- Blog, Medium, LinkedIn, Twitter/X, Threads
    |-- All as DRAFTS — never auto-publish
    |-- Scheduled publishing via cron
    |
    v
[Publish Log -> feedback into Scorer]
```

---

## 10. Configuration Points

To adapt this system to your niche, customize:

1. **Pillars** — define your content categories and target weights per growth phase
2. **Awareness levels** — adjust the 3 levels to match your audience's journey
3. **Sources** — plug in your own topic sources (subreddits, newsletters, CRM, etc.)
4. **Style profiles** — create your baseline voice + reference creator profiles
5. **Style config** — map pillar x awareness to style tags
6. **Script rules** — adjust word count limits, language mix, energy markers
7. **Distribution platforms** — add/remove platforms and their constraints
8. **Cron schedule** — set harvest and publish times for your timezone
9. **Voice rules** — define banned words, tone guidelines, formatting rules
10. **Scoring weights** — tune the 5 signals to prioritize what matters for your content

---

## 11. Environment Variables

```env
# VPS
VPS_HOST=
VPS_PORT=9090

# Telegram
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

# Google Sheets (if using sheet-based sources)
GOOGLE_SHEETS_ID=
GOOGLE_SHEETS_CREDENTIALS=

# Dashboard Auth
DASHBOARD_USERNAME=
DASHBOARD_PASSWORD=

# Vault
VAULT_PATH=
VAULT_GIT_REMOTE=
```

---

## 12. Key Rules

### Pipeline Rules
- Never skip a step — every topic goes through all phases
- Never fabricate data — report errors honestly
- Never auto-publish — human approval required at pick step and publish step
- If evaluator fails/times out: default to V1 with note
- Scripts saved to vault BEFORE notifications sent
- Topic database is single source of truth

### Script Quality Rules
- Max 100 words (aim 70-80) for teleprompter
- Min 3 action steps (practical, doable today)
- Min 1 relatable analogy per variation
- Min 1 personal authority line
- Min 1 active CTA with question (not passive)
- Min 2 energy modulation markers in teleprompter
- Triple alignment: visual hook + text hook + spoken hook must match

### Voice Quality Rules
- Conversational, not telegraphic
- CAPS for emphasis only (not every other word)
- Preserve natural fillers (intentional voice markers)
- No jargon, no hype, no coach-speak
- One idea per line
