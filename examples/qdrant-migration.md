# Qdrant Migration Guide

> **For new deployments** using the [mem0-vigil](https://github.com/jamebobob/mem0-vigil) fork, migration scripts are included in the fork repo. This guide is primarily useful for upgrading existing single-pool or two-pool setups to per-group isolation.

If you're adding multi-pool routing to an existing Qdrant-backed Mem0 setup,
you need to tag your existing memories so the pool routing can find them.

## The Problem

Before multi-pool, all memories live in a single Qdrant collection with no
pool metadata. After multi-pool, the routing logic filters by `user_id`
(which becomes the pool key). Untagged memories work fine for pool routing
(they already have the correct `user_id`), but they lack the provenance
metadata that new memories get automatically.

The migration adds provenance tags and a rollback marker to all existing points.

## Quick Start

See [mem0-vigil](https://github.com/jamebobob/mem0-vigil)
for standalone migration scripts with CLI arguments.

```bash
# Create payload indexes (run once)
python3 create-indexes.py

# Tag all existing memories (run once, before enabling the patch)
python3 migrate-memories.py

# Verify
python3 migrate-memories.py --dry-run
```

## What the Migration Does

Adds five fields to every existing Qdrant point:

| Field | Value | Purpose |
|-------|-------|---------|
| `is_private` | `true` | Conservative default. Pre-social-agent memories are all private. |
| `source_channel` | `"unknown"` | Can't determine retroactively. New memories get the real channel. |
| `conversation_type` | `"dm"` | Pre-social-agent memories are all from DM context. |
| `agent_id` | `"main"` | Pre-social-agent memories are all from the main agent. |
| `migration_state` | `"pre_multipool_tagged"` | Rollback marker. Identifies migrated points. |

The migration is additive. It doesn't modify existing data, vectors, or the
`userId` field that Mem0 uses for pool routing.

## Indexes

Five payload indexes enable efficient filtered queries:

| Field | Type | Purpose |
|-------|------|---------|
| `is_private` | bool | Privacy flag filtering |
| `conversation_type` | keyword | DM vs group vs cron |
| `source_channel` | keyword | Telegram, WhatsApp, etc. |
| `chat_id` | keyword | Per-chat filtering |
| `agent_id` | keyword | Per-agent filtering |

These aren't strictly required for pool routing (which uses `user_id`), but
they enable provenance-based queries and future trust scoring.

## Rollback

The `migration_state: "pre_multipool_tagged"` marker lets you identify and
revert all migrated points. See the rollback script in
[mem0-vigil](https://github.com/jamebobob/mem0-vigil).

This removes all five added fields. It doesn't undo pool routing config
changes or delete any memories.

## Timing

Run the migration **before** enabling the multi-pool patch. New memories
created after the patch will be tagged automatically by the pool routing
hooks with accurate provenance (real channel, real conversation type, real
agent ID).
