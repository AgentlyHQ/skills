# Quick Start Guide

## 30-Second Setup

```bash
# 1. Install gws CLI
npm install -g @googleworkspace/cli

# 2. Authenticate
gws auth login

# 3. Done!
```

## How It Works

**Step 1: Ingest** (do once)
```
You: "Ingest my meeting notes from 'Team Meetings'"

Claude:
→ Finds folder in Google Drive
→ Exports all documents as text
→ Parses and structures content
→ Builds searchable knowledge bank
→ ✓ Ready!
```

**Step 2: Query** (do many times)
```
You: "What was decided about API redesign?"

Claude:
→ Searches local knowledge bank
→ Reads relevant meeting(s)
→ Extracts answer
→ Cites source with date
```

## First Time Use

```
# 1. Ingest your meetings
"Ingest my meeting notes from 'Team Meetings' folder"

# 2. Wait for confirmation
"✓ Knowledge bank built with 23 meeting notes"

# 3. Start asking questions
"What did we discuss about hiring?"
"What are my action items?"
"Summarize Q2 roadmap discussions"
```

## What You Can Ask

### Specific Questions
```
"What was decided about [topic]?"
"Who is responsible for [project]?"
"What's the timeline for [initiative]?"
```

### Searches
```
"Find all discussions about [topic]"
"Show me mentions of [person]"
"What meetings covered [subject]?"
```

### Summaries
```
"Summarize all [topic] discussions"
"What's the history of [decision]?"
"How did [topic] evolve over time?"
```

### Action Items
```
"What are my action items?"
"What tasks are assigned to [person]?"
"What's due this week?"
```

## Knowledge Bank Location

All meetings stored at: `/tmp/knowledge_bank/`

```
/tmp/knowledge_bank/
├── INDEX.md              # List of all meetings
├── *.txt                 # Raw meeting content
└── parsed/
    └── *.json           # Structured data
```

## Key Commands

| Command | What It Does |
|---------|--------------|
| `"Ingest from [folder]"` | Build knowledge bank |
| `"What was decided about X?"` | Query knowledge bank |
| `"Show knowledge bank status"` | Check what's loaded |
| `"Refresh knowledge bank"` | Update with new meetings |

## Example Session

```
┌──────────────────────────────────────────┐
│ You: Ingest my meeting notes from        │
│      'Q1 2026 Meetings' folder           │
└──────────────────────────────────────────┘
                  ↓
┌──────────────────────────────────────────┐
│ Claude: [Runs gws commands]              │
│                                          │
│ ✓ Found 18 meeting notes                │
│ ✓ Exported all documents                │
│ ✓ Parsed and structured                 │
│ ✓ Built searchable index                │
│                                          │
│ Knowledge bank ready with 18 meetings!  │
│ You can now ask me questions.           │
└──────────────────────────────────────────┘
                  ↓
┌──────────────────────────────────────────┐
│ You: What was decided about the mobile   │
│      app redesign?                       │
└──────────────────────────────────────────┘
                  ↓
┌──────────────────────────────────────────┐
│ Claude: [Searches knowledge bank]        │
│         [Reads relevant meetings]        │
│         [Extracts answer]                │
│                                          │
│ **Answer**: Team decided to prioritize  │
│ mobile app redesign as Q2 priority #1    │
│                                          │
│ **Source**: Product Planning -           │
│ February 20, 2026                        │
│                                          │
│ **Details**: Mobile gets 60% of         │
│ engineering resources. Launch target     │
│ is June 30, 2026.                       │
└──────────────────────────────────────────┘
```

## Troubleshooting

### "Knowledge bank not found"
**Fix**: Run ingestion first
```
"Ingest my meeting notes from [folder name]"
```

### "No meetings found about X"
**Fix**: Try different keywords
```
# Instead of "API v2"
Try: "API" or "redesign" or "new API"
```

### "gws command not found"
**Fix**: Install gws CLI
```bash
npm install -g @googleworkspace/cli
gws auth login
```

## Tips

1. **Organize folder**: Keep meeting notes in one folder
2. **Name consistently**: "Meeting Type - YYYY-MM-DD"
3. **Refresh regularly**: Re-ingest when new meetings added
4. **Be specific**: Better questions get better answers

## What Makes This Different

Traditional approach:
```
Question → Search Google Drive → Read snippets → Answer
(Slow, incomplete, repeated API calls)
```

This approach:
```
Ingest once: Google Drive → Local knowledge bank

Then for every question:
Question → Search local files → Read full content → Answer
(Fast, complete, no API calls)
```

## Storage

- **Location**: `/tmp/knowledge_bank/`
- **Persistence**: Session-only (need to re-ingest after restart)
- **Size**: ~1-5KB per meeting note
- **Format**: Plain text + JSON

## Ready?

Just say:
```
"Ingest my meeting notes from [your folder name]"
```

Then start asking questions!

---

**Pro tip**: After ingestion, check what was loaded:
```
"Show me the knowledge bank index"
```

This shows all meetings that were ingested and are available for querying.
