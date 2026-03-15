# Qdrant Migration Guide

If you're adding multi-pool routing to an existing Qdrant-backed Mem0 setup,
you need to tag your existing memories so the pool routing can find them.

## The Problem

Before multi-pool, all memories live in a single Qdrant collection with no
pool metadata. After multi-pool, the routing logic filters by `is_private`
and pool tags. Untagged memories are invisible to pool-aware queries.

## Migration Strategy

Tag all existing memories as private (conservative default). They were all
created in DM context before the social agent existed, so `is_private: true`
is factually correct.

### Step 1: Create Payload Indexes

```bash
# These indexes enable efficient filtering on pool-related fields
curl -X PUT "http://localhost:6333/collections/YOUR_COLLECTION/index" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "is_private", "field_schema": "bool"}'

curl -X PUT "http://localhost:6333/collections/YOUR_COLLECTION/index" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "conversation_type", "field_schema": "keyword"}'

curl -X PUT "http://localhost:6333/collections/YOUR_COLLECTION/index" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "source_channel", "field_schema": "keyword"}'

curl -X PUT "http://localhost:6333/collections/YOUR_COLLECTION/index" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "chat_id", "field_schema": "keyword"}'

curl -X PUT "http://localhost:6333/collections/YOUR_COLLECTION/index" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "agent_id", "field_schema": "keyword"}'
```

### Step 2: Tag Existing Memories

Use Qdrant's scroll + set_payload to tag all existing points:

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)
collection = "YOUR_COLLECTION"

# Scroll through all points and tag them
offset = None
tagged = 0

while True:
    results = client.scroll(
        collection_name=collection,
        limit=100,
        offset=offset,
        with_payload=False,
        with_vectors=False,
    )
    points, next_offset = results

    if not points:
        break

    ids = [p.id for p in points]

    client.set_payload(
        collection_name=collection,
        payload={
            "is_private": True,
            "source_channel": "dm",
            "conversation_type": "dm",
            "agent_id": "main",
            "migration_state": "pre_multipool_tagged",
        },
        points=ids,
    )

    tagged += len(ids)
    offset = next_offset

    if next_offset is None:
        break

print(f"Tagged {tagged} memories as private")
```

### Step 3: Verify

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue

# Count tagged memories
count = client.count(
    collection_name=collection,
    count_filter=Filter(
        must=[
            FieldCondition(key="is_private", match=MatchValue(value=True)),
            FieldCondition(key="migration_state", match=MatchValue(value="pre_multipool_tagged")),
        ]
    ),
)
print(f"Tagged memories: {count.count}")
```

## Rollback

The `migration_state: "pre_multipool_tagged"` field lets you identify and
revert migrated memories if needed:

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, FilterSelector

# Remove migration tags (revert to pre-migration state)
# Note: this removes the tags but doesn't undo pool routing config changes
client.delete_payload(
    collection_name=collection,
    keys=["is_private", "migration_state"],
    points_selector=FilterSelector(
        filter=Filter(
            must=[
                FieldCondition(key="migration_state", match=MatchValue(value="pre_multipool_tagged")),
            ]
        )
    ),
)
```

## Notes

- Run the migration BEFORE enabling the multi-pool patch. Untagged memories
  won't be found by pool-aware recall queries.
- The migration is additive (adds payload fields). It doesn't modify existing
  data or vectors.
- New memories created after the patch will be tagged automatically by the
  pool routing hooks.
