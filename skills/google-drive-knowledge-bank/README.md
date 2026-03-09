# Google Drive Knowledge Bank

Build a searchable, queryable knowledge base from your Google Drive meeting notes.

## What This Does

This skill is a **two-phase system**:

### Phase 1: Ingestion
- Pulls all meeting notes from a Google Drive folder
- Exports them as text files
- Parses and structures the content
- Builds a searchable index
- Stores everything locally as a knowledge bank

### Phase 2: Querying
- Searches the stored knowledge bank (NOT Google Drive)
- Reads full meeting content when relevant
- Answers questions grounded in actual notes
- Never hallucinates or makes up information

## Why This Approach?

**Traditional approach**: Search Google Drive every time → Slow, repeated API calls

**This approach**: Ingest once → Query locally many times → Fast, accurate

## Prerequisites

```bash
# Google Workspace CLI
npm install -g @googleworkspace/cli

# Authenticate
gws auth login

# Verify
gws drive files list --max-results 5
```

## Quick Start

### 1. Ingest Your Meeting Notes

```
You: "Ingest my meeting notes from the 'Team Meetings' folder"

Claude: [Uses gws to pull all notes, parse them, build knowledge bank]
✓ Knowledge bank built with 23 meeting notes
✓ Ready to answer questions!
```

### 2. Ask Questions

```
You: "What was decided about the API redesign?"

Claude: [Searches local knowledge bank, reads relevant meetings, answers]

**Answer**: Team decided on REST API for v2 (not GraphQL) due to client compatibility.

**Source**: Engineering Sync - February 15, 2026

[Full context from meeting notes]
```

## Complete Example

```bash
# ═══════════════════════════════════════════════════
# INGESTION PHASE
# ═══════════════════════════════════════════════════

$ gws drive files search --query "name = 'Meeting Notes'"
# Find folder ID

$ gws drive files list --parent-id "FOLDER_ID"
# List all documents in folder

$ mkdir -p /tmp/knowledge_bank/parsed
# Create storage

$ for each document:
    gws drive files export --file-id "..." --output-file "/tmp/knowledge_bank/FILE.txt"
    # Parse and structure
    # Build searchable index
  done

✓ Knowledge bank ready with N meetings

# ═══════════════════════════════════════════════════
# QUERY PHASE
# ═══════════════════════════════════════════════════

User asks: "What are my action items?"

$ grep -l "action\|@username\|TODO" /tmp/knowledge_bank/*.txt
# Find meetings with action items

$ for each match:
    cat /tmp/knowledge_bank/parsed/FILE.json
    # Read full content
    # Extract action items
    # Cite source
  done

Returns: List of action items with sources
```

## Knowledge Bank Structure

After ingestion:

```
/tmp/knowledge_bank/
├── metadata.txt                    # File mappings
├── INDEX.md                        # Human-readable index
├── file_id_1.txt                   # Raw meeting content
├── file_id_2.txt
├── file_id_3.txt
└── parsed/
    ├── file_id_1.json             # Structured data
    ├── file_id_2.json
    └── file_id_3.json
```

Each JSON contains:
```json
{
  "file_id": "abc123",
  "filename": "Engineering Sync - 2026-03-01",
  "meeting_date": "2026-03-01",
  "modified_time": "2026-03-01T10:30:00Z",
  "content": "full meeting notes text...",
  "word_count": 1523,
  "indexed_at": "2026-03-09T14:00:00Z"
}
```

## Usage Patterns

### Single Decision Lookup
```
Q: "What was decided about pricing?"
A: [Searches "pricing" → Reads matching meetings → Extracts decision → Cites source]
```

### Timeline/Evolution
```
Q: "How did the API discussion evolve over time?"
A: [Finds all "API" mentions → Sorts by date → Shows progression → Cites each meeting]
```

### Action Items
```
Q: "What are my open action items?"
A: [Searches "action|@username|TODO" → Extracts items → Groups by meeting → Lists with due dates]
```

### Topic Summary
```
Q: "Summarize all hiring discussions"
A: [Finds "hiring" in all meetings → Reads each → Synthesizes key points → Cites sources]
```

## Key Features

✅ **Fast**: Query local files, not Google Drive API  
✅ **Accurate**: Always reads full meeting content  
✅ **Grounded**: Never makes up information  
✅ **Cited**: Every answer includes source and date  
✅ **Comprehensive**: Can synthesize across multiple meetings  

## Commands

### Ingest
```
"Ingest my meeting notes from [folder name]"
"Load meeting notes from folder ID: [id]"
"Build knowledge bank from Team Meetings folder"
```

### Query
```
"What was decided about X?"
"Find all discussions about Y"
"What are action items from engineering syncs?"
"Summarize Q2 roadmap discussions"
"Show me the March 1st meeting notes"
```

### Maintenance
```
"Refresh the knowledge bank"
"Show knowledge bank status"
"How many meetings are indexed?"
```

## Workflow

```
┌─────────────────────────────────────────────┐
│         PHASE 1: INGESTION                  │
│                                             │
│  Google Drive Folder                        │
│       │                                     │
│       ├─ Meeting 1.docx ───┐               │
│       ├─ Meeting 2.docx ───┤               │
│       └─ Meeting 3.docx ───┤               │
│                             │               │
│                             ↓               │
│                        gws export           │
│                             │               │
│                             ↓               │
│                    /tmp/knowledge_bank/     │
│                    ├─ Meeting1.txt          │
│                    ├─ Meeting2.txt          │
│                    ├─ Meeting3.txt          │
│                    └─ parsed/*.json         │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         PHASE 2: QUERYING                   │
│                                             │
│  User Question                              │
│       │                                     │
│       ↓                                     │
│  Search local knowledge bank                │
│       │                                     │
│       ├─ grep for keywords                  │
│       ├─ find matching meetings             │
│       └─ read full content                  │
│             │                               │
│             ↓                               │
│       Extract answer                        │
│             │                               │
│             ↓                               │
│       Cite sources                          │
│             │                               │
│             ↓                               │
│       Return answer                         │
└─────────────────────────────────────────────┘
```

## Benefits Over Real-Time Search

| Aspect | Real-Time Search | Knowledge Bank |
|--------|------------------|----------------|
| Speed | Slow (API calls every time) | Fast (local files) |
| Cost | High (repeated API usage) | Low (ingest once) |
| Consistency | May change between queries | Stable snapshot |
| Synthesis | Hard (multiple API calls) | Easy (local access) |
| Accuracy | Limited by API snippets | Full content always |

## Limitations

- Knowledge bank stored in `/tmp/` (session-only)
- Must re-ingest to get new meetings
- Requires disk space for all meeting notes
- No real-time updates (by design)

## Best Practices

1. **Organize in Drive**: Keep meeting notes in a dedicated folder
2. **Consistent naming**: Use "Meeting Type - YYYY-MM-DD" format
3. **Structured notes**: Include sections like "Decisions", "Action Items"
4. **Regular refresh**: Re-ingest weekly or when important meetings added
5. **Clear ingestion**: Always tell user when ingestion is complete

## Error Handling

### No knowledge bank
```
⚠ Knowledge bank not initialized.
Please run: "Ingest my meeting notes from [folder]"
```

### No matches
```
No meetings found discussing "keyword"

Try:
  - Alternative search terms
  - Broader keywords
  - Checking available meetings: cat /tmp/knowledge_bank/INDEX.md
```

### Incomplete info
```
"I found partial information in [meeting], but the discussion 
appears incomplete. The notes mention X but don't include Y."
```

## Philosophy

This skill is **pull-based, not push-based**:

1. **Pull** meeting notes into knowledge bank (once)
2. **Query** knowledge bank (many times)
3. **Refresh** when needed (occasionally)

This makes it fast, reliable, and accurate for answering questions about historical meetings.

---

**Ready to start?**

```
"Ingest my meeting notes from the 'Team Meetings' folder"
```
