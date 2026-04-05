# BEAST Content Pipeline

A multi-agent content production system that sources topics, scores them for virality, generates Tamil business education scripts through 7 AI agents, and distributes English adaptations across 5 platforms.

## Architecture

```
TopicMaster (sourcing + scoring)
    |
    v
BEAST Pipeline (7 agents)
    |
    v
Distributor (5-platform content waterfall)
```

## Quick Start

See [PIPELINE.md](PIPELINE.md) for the full system spec — covering topic sourcing, scoring algorithm, all 7 agents, database schema, API endpoints, and infrastructure.

## Stack

- **Runtime:** Node.js on VPS (PM2 managed)
- **Database:** SQLite (`topic_brain.db`, `queue.db`)
- **AI:** Claude CLI (`claude -p`) for all content generation (zero API cost)
- **AI (lightweight):** Claude Haiku for classification, clustering, scoring
- **Dashboard:** Express.js, port 9090
- **Vault:** Obsidian (git-synced to VPS)
- **Notifications:** Telegram bot

## License

Private — not for redistribution.
