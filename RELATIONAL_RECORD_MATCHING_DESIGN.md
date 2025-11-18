# Cognee Relational Record Matching - MVP v1 Design

## Executive Summary

Extend Cognee's knowledge graph entity resolution to link extracted entities to existing relational database records (orgs, people). Entities matched to DB records are enriched with metadata linking back to the authoritative record; unmatched entities remain in the graph without corresponding relational records.

**Phase**: Integrated into `expand_with_nodes_and_edges()` after ontology resolution

---

## High-Level Plan

### Pipeline Flow

```
Extract Nodes/Edges
  ↓
Ontology Resolution (validate entity type against ontology)
  ↓
[NEW] Relational Record Matching (link entity to DB record if exists)
  ↓
Entity Enrichment (add DB record metadata to payload)
  ↓
Deduplication & Storage in GraphDB
```

### Architecture

**Three new components:**

1. **RelationalRecordMatcher** - Core fuzzy matching logic
   - Accepts: entity name, ontology-resolved type
   - Returns: Optional link to DB record with confidence score
   - Uses: Existing FuzzyMatchingStrategy with configurable threshold

2. **RelationalMatchConfig** - Configuration object
   - Maps ontology classes to relational tables and match fields
   - Example: `Organization` → `(orgs, name)`
   - Example: `Person` → `(people, full_name)`

3. **RelationalEntityLink** - Data model for entity-to-record link
   - Stores: record ID, table name, match field, confidence score
   - Serializes to DataPoint payload

**Integration point:** `expand_with_nodes_and_edges()` in `cognee/modules/graph/utils/`

---

## Identified Ambiguities & Resolution Options

### Ambiguity #1: Record Lookup Scope & Query Strategy

**Question:** How should we fetch records from orgs/people tables?

**Options:**

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| **A: Batch load all** | Load all orgs/people into memory at start of cognify | Fast matching (no DB queries per entity), simple | High memory if tables large |
| **B: Paginated batch** | Load records in chunks (e.g., 1000 at a time) | Memory efficient, still fast | More complex code |
| **C: Individual queries** | Query DB for each entity match attempt | Minimal memory | Slow (N queries), scales poorly |
| **D: No search** | Require exact config of record IDs upfront | Simple | No automatic discovery |

**Best Option(s):** **A (Batch load all)** for MVP v1
- Rationale: Typical org/people tables < 100K records; single batch load is simplest
- Fallback: If production data proves too large, migrate to B

---

### Ambiguity #2: Match Field Selection

**Question:** Which field should we fuzzy-match against for each table?

**Options:**

| Entity Type | Orgs Table | People Table |
|-------------|-----------|--------------|
| **Organization** | (A) `name` only | N/A |
| | (B) `name` + `website` | |
| **Person** | N/A | (A) `full_name` only |
| | | (B) `full_name` + `primary_email` |
| | | (C) All text fields (first_name, last_name, username, etc.) |

**Best Option(s):** **Primary field only** (Orgs: `name`, People: `full_name`)
- Rationale: MVP should be simple and predictable
- Future: Add secondary field fallback if needed

---

### Ambiguity #3: Fuzzy Match Confidence Threshold

**Question:** What similarity score (0.0-1.0) is required to accept a match?

**Options:**

| Threshold | Matching Examples | Pros | Cons |
|-----------|-------------------|------|------|
| **0.75** | "Apple Inc" ↔ "Apple Incs" | High recall, catches typos | False positives possible |
| **0.80** | "Apple Inc" ↔ "Apple Inc." | Balanced (current Cognee default) | May miss some typos |
| **0.85** | "Apple Inc" ↔ "Apple Inc." | Conservative | May miss legitimate matches |
| **0.95** | "Apple Inc" ↔ "Apple Inc" | Only near-exact | Strict, may underutilize DB |

**Best Option(s):** **0.85**
- Rationale: Conservative for MVP; favor correctness over recall
- Matching strategy already exists at this threshold in ontology resolver
- Can be tuned upward if false positives occur

---

### Ambiguity #4: Multiple Candidate Matches

**Question:** If multiple records have same/similar fuzzy match score, what do we do?

**Options:**

| Option | Behavior | Risk |
|--------|----------|------|
| **A: Highest score only** | If tie, pick first; log warning | May pick wrong record in tie case |
| **B: Reject on tie** | If multiple matches with similar score, don't match | Conservative but safe; may underutilize matches |
| **C: Human review** | Surface ambiguity for manual review | Too complex for MVP |
| **D: Exact match only** | Only accept if name exactly matches (after normalization) | Safe but loses fuzzy benefit |

**Best Option(s):** **A (Highest score only)** with note that ties are rare
- Rationale: Fuzzy matching rarely produces true ties; when it does, order is stable
- Simplest for MVP
- If tie occurs (e.g., score 0.92), pick first result consistently

---

### Ambiguity #5: Entity Type to Table Mapping

**Question:** How do we know which table to search for each entity type?

**Options:**

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| **A: Hard-coded mapping** | `Organization` → `orgs`, `Person` → `people` | Simple, no config needed | Inflexible for schema changes |
| **B: Config object** | Customizable dict: `entity_type_mapping` | Flexible, can be overridden | Requires more setup |
| **C: Search all tables** | For each entity, try all table types | Maximum coverage | Slow, unclear precedence |
| **D: Ontology annotation** | RDF ontology includes table hints | Semantic purity | Requires ontology changes |

**Best Option(s):** **B (Config object) with sensible defaults**
- Rationale: Flexible but defaults work for common types
- Allow override at `cognify()` call time
- MVP defaults:
  - `Organization` → `(orgs, name)`
  - `Company` → `(orgs, name)`
  - `Person` → `(people, full_name)`
  - `Individual` → `(people, full_name)`

---

### Ambiguity #6: Enrichment of Matched Entities

**Question:** What metadata should we add to matched entities?

**Options:**

| Option | Metadata Added | Pros | Cons |
|--------|---|---|---|
| **A: Link only** | `postgres_record_id`, `postgres_table`, `match_confidence` | Minimal, lean | Entity name may differ from DB |
| **B: Full enrichment** | + `canonical_name` from DB, matched field, all DB fields | Complete info | Larger payload, more coupling |
| **C: Record reference** | Store full DB record object in payload | Easy access | Violates separation of concerns |
| **D: No enrichment** | Just mark `db_matched: true` | Simplest | No way to track which record matched |

**Best Option(s):** **A (Link only) + use canonical name from DB**
- Rationale: Store minimal metadata; canonical name is critical for deduplication
- Payload structure:
  ```python
  {
    "relational_record": {
      "postgres_record_id": "550e8400...",
      "postgres_table": "orgs",
      "match_field": "name",
      "match_confidence": 0.92
    },
    "canonical_name": "Apple Inc."  # From DB record
  }
  ```
- Entity name is replaced with DB's canonical name to ensure consistency

---

### Ambiguity #7: Unmatched Entities Handling

**Question:** What happens to entities that don't match any DB record?

**Options:**

| Option | Behavior | Graph State | Relational State |
|--------|----------|-------------|------------------|
| **A: Float in graph** | Create DataPoint with `relational_record: null` | Entity exists | No record |
| **B: Exclude from graph** | Don't create DataPoint if no match | Nothing | Nothing |
| **C: Create stub record** | Auto-create record in orgs/people table | Entity exists | Auto-created stub |
| **D: Defer to queue** | Mark for manual review; don't store yet | Temporary | Waiting |

**Best Option(s):** **A (Float in graph)**
- Rationale: Graph still captures knowledge; no automatic mutations to source DB
- Supports future manual curation workflow
- Keeps graph complete while maintaining separation of concerns
- Mark with `db_matched: false` for later filtering if needed

---

### Ambiguity #8: Secondary Field Fallback

**Question:** Should we try additional fields if primary match fails?

**Options:**

| Option | Example | Pros | Cons |
|--------|---------|------|------|
| **A: No fallback** | Person: only `full_name`, stop | Simple, predictable | May miss matches |
| **B: One fallback** | Person: try `full_name`, then `primary_email` | Reasonable coverage | Slightly more complex |
| **C: Multiple fallback** | Person: name → email → username → linkedin_id | Comprehensive | Complex, unclear precedence |

**Best Option(s):** **A (No fallback)** for MVP v1
- Rationale: KISS principle; primary field matching usually sufficient
- If needed for production, can add as v1.1 feature
- Prevents confusion about match precedence

---

### Ambiguity #9: Multi-Tenant & Access Control

**Question:** Should matching respect user/org context?

**Options:**

| Option | Scope | Pros | Cons |
|--------|-------|------|------|
| **A: Global search** | Match against all records in all orgs | Covers all data | May match across orgs unintended |
| **B: Current org only** | Only match records in user's current org | Secure, scoped | May miss legitimate matches |
| **C: User's visible orgs** | Match against orgs user has access to | Balanced | Complex to implement |

**Best Option(s):** **A (Global search)** for MVP v1
- Rationale: Matching is non-mutating (read-only); no security risk
- Ensures maximum entity resolution coverage
- Access control applied at API layer (when querying/modifying matched records)
- Note: Graph nodes marked with matched record IDs; access controlled elsewhere

---

### Ambiguity #10: Reverse Linking (Postgres → Graph)

**Question:** Should DB records know about matched graph entities?

**Options:**

| Option | Direction | Implementation | Pros | Cons |
|--------|-----------|---|---|---|
| **A: One-way** | Graph → DB only | Store `postgres_record_id` in DataPoint | Clean separation | Can't query "which entities link to this record" |
| **B: Two-way** | Graph ↔ DB | Add FK/reference in orgs/people tables | Can query both directions | Requires schema changes |
| **C: Join table** | Via pivot table | Create `entity_to_record` mapping table | Explicit relationships | More complex, more storage |

**Best Option(s):** **A (One-way)** for MVP v1
- Rationale: No schema changes to orgs/people tables needed
- Graph is overlay; DB remains independent
- If needed later (e.g., "which graph entities reference this org?"), can add reverse index

---

## MVP v1 Configuration

### Environment Variables (Optional)

```bash
# Enable relational matching (default: true if matcher provided)
RELATIONAL_MATCH_ENABLED=true

# Fuzzy match threshold (default: 0.85)
RELATIONAL_MATCH_CONFIDENCE_THRESHOLD=0.85
```

### Programmatic Configuration (In Code)

```python
RelationalMatchConfig(
    confidence_threshold=0.85,
    entity_type_mapping={
        "Organization": ("orgs", "name"),
        "Company": ("orgs", "name"),
        "Person": ("people", "full_name"),
        "Individual": ("people", "full_name"),
    }
)
```

---

## Summary Table: MVP v1 Decisions

| Ambiguity | Decision | Rationale |
|-----------|----------|-----------|
| Record lookup | Batch load all | Simple, fast for typical table sizes |
| Match field | Primary only (name/full_name) | MVP simplicity |
| Confidence threshold | 0.85 | Conservative, matches Cognee default |
| Multiple matches | Highest score wins | Rare ties; stable ordering |
| Entity-to-table mapping | Config object with defaults | Flexible but sensible defaults |
| Enrichment strategy | Link metadata + canonical name | Minimal coupling, essential info only |
| Unmatched entities | Float in graph | Supports future curation |
| Secondary fields | No fallback | Keep MVP simple |
| Multi-tenant scope | Global search | Read-only operation; secure at API layer |
| Reverse linking | One-way (Graph→DB) | No schema changes needed |

---

## Integration Points

### Modified Files
- `cognee/modules/graph/utils/expand_with_nodes_and_edges.py`
  - Add `relational_record_matcher` parameter
  - Call matcher after ontology resolution, before deduplication

### New Files
- `cognee/modules/graph/utils/relational_record_matcher.py` - Core matcher
- `cognee/modules/graph/models/relational_match_config.py` - Configuration
- `cognee/modules/graph/models/relational_entity_link.py` - Data model
- `cognee/infrastructure/databases/relational/queries/record_batch_queries.py` - DB fetching

### Affected Components
- DataPoint payload: Add optional `relational_record` dict and `canonical_name` field
- `cognify()` method: Accept optional `relational_record_matcher` parameter

---

## Example: End-to-End Flow

**Input:** Extracted entity
```python
node = Node(
    id="apple_entity",
    name="Apple Inc",
    type="Organization",
    description="Tech company"
)
```

**Process:**
1. Ontology resolver: `Organization` → found in ontology ✓
2. Relational matcher:
   - Load all orgs: ~500 records
   - Fuzzy match "Apple Inc" against org names
   - Find: `orgs.id = "550e8400..."`, `name = "Apple Inc."`
   - Score: 0.95 > 0.85 ✓
3. Enrichment:
   - Name: "Apple Inc" → "Apple Inc." (DB canonical)
   - Add metadata to payload

**Output:** Enhanced node
```python
node.payload = {
    "canonical_name": "Apple Inc.",
    "ontology_class": "Organization",
    "ontology_valid": True,
    "db_matched": True,
    "relational_record": {
        "postgres_record_id": "550e8400-...",
        "postgres_table": "orgs",
        "match_field": "name",
        "match_confidence": 0.95
    }
}
```

---

## Future Enhancements (Not MVP)

- Secondary field fallback (email for people, website for orgs)
- People-org relationship matching (check `people_orgs` junction)
- Phonetic/semantic matching (Soundex, embeddings)
- Confidence adjustment based on field type
- Reverse index: "which entities reference this org?"
- Batch matching metrics & observability
