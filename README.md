# Multi-Agent Content Pipeline

A multi-agent content production system that sources topics, scores them for virality, generates scripts through 7 AI agents, and distributes adaptations across multiple platforms.

Built to run entirely on Claude CLI (`claude -p`) — zero API cost on Max plan.

## Architecture

```
TopicMaster (sourcing + scoring)
    |
    v
Content Pipeline (7 agents)
    |
    v
Distributor (multi-platform content waterfall)
```

## Quick Start

See [PIPELINE.md](PIPELINE.md) for the full system spec — covering topic sourcing, scoring algorithm, all 7 agents, database schema, API endpoints, and infrastructure.

## Stack

- **Runtime:** Node.js on VPS (PM2 managed)
- **Database:** SQLite
- **AI:** Claude CLI (`claude -p`) for all content generation
- **AI (lightweight):** Claude Haiku for classification, clustering, scoring
- **Dashboard:** Express.js
- **Vault:** Obsidian (git-synced to VPS)
- **Notifications:** Telegram bot

## Customization

The system is designed around pluggable content pillars, awareness levels, style profiles, and distribution platforms. See the `## Configuration Points` section in PIPELINE.md to adapt it to your niche.

## License

MIT
