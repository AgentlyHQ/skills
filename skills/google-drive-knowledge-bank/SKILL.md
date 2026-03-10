---
name: google-drive-knowledge-bank
description: Build and query a knowledge bank from Google Docs meeting notes. Use when the user wants to ingest meeting notes from Google Drive, or when answering questions about previously ingested meeting notes. Trigger on phrases like "ingest my meeting notes", "load meeting notes from folder", "what did we discuss about X" (after ingestion), "find information about Y in meeting notes", "build knowledge base from meetings". Always use this skill when working with a corpus of meeting notes that needs to be searchable.
compatibility:
  - Google Workspace CLI (gws) - https://github.com/googleworkspace/cli
  - Authenticated Google Workspace account with Drive access
license: MIT
metadata:
  author: Agently
  version: 1.0.0
---

# Meeting Notes Knowledge Bank

Transform a folder of Google Docs meeting notes into a queryable knowledge bank. This skill ingests meeting notes, parses them into structured format, and stores them for fast, accurate retrieval when answering questions.

## Two-Phase Workflow

### Phase 1: Ingestion (Build the Knowledge Bank)
Pull meeting notes from Google Drive, parse their content, and store as structured data.

### Phase 2: Querying (Answer Questions)
Search the stored knowledge bank to answer questions accurately, always grounding responses in the actual meeting content.

---

## Prerequisites

This skill requires the Google Workspace CLI (`gws`):

```bash
# Check if installed
gws --version

# If not installed
npm install -g @googleworkspace/cli

# Authenticate
gws auth login

# Verify access
gws drive files list --max-results 5
```

---

## PHASE 1: INGESTION

### Step 1: Get Folder ID from User

Ask the user for their meeting notes folder. They can provide:
- Folder name (you'll search for it)
- Folder ID directly
- Folder URL (extract ID from it)

**Find folder by name:**
```bash
gws drive files search \
  --query "name = 'Meeting Notes' and mimeType = 'application/vnd.google-apps.folder'" \
  --format json
```

**Extract folder ID from URL:**
URL format: `https://drive.google.com/drive/folders/FOLDER_ID`

### Step 2: List All Meeting Notes in Folder

```bash
# Get all Google Docs in the folder
gws drive files list \
  --parent-id "FOLDER_ID" \
  --query "mimeType = 'application/vnd.google-apps.document'" \
  --format json > /tmp/meeting_files_list.json

# Parse to see what we got
cat /tmp/meeting_files_list.json | jq -r '.files[] | "\(.name) | \(.id) | \(.modifiedTime)"'
```

### Step 3: Export All Meeting Notes

```bash
# Create knowledge bank directory
mkdir -p /tmp/knowledge_bank

# Parse file list and export each document
FILE_IDS=($(cat /tmp/meeting_files_list.json | jq -r '.files[].id'))

for file_id in "${FILE_IDS[@]}"; do
  # Get file metadata
  FILE_NAME=$(cat /tmp/meeting_files_list.json | jq -r ".files[] | select(.id == \"$file_id\") | .name")
  MODIFIED=$(cat /tmp/meeting_files_list.json | jq -r ".files[] | select(.id == \"$file_id\") | .modifiedTime")
  
  # Export as plain text
  echo "Ingesting: $FILE_NAME"
  gws drive files export \
    --file-id "$file_id" \
    --mime-type "text/plain" \
    --output-file "/tmp/knowledge_bank/${file_id}.txt"
  
  # Store metadata
  echo "$file_id|$FILE_NAME|$MODIFIED" >> /tmp/knowledge_bank/metadata.txt
done

echo "✓ Ingested ${#FILE_IDS[@]} meeting notes"
```

### Step 4: Parse and Structure Content

For each meeting note, extract structured information:

```bash
# Create parsed knowledge base
mkdir -p /tmp/knowledge_bank/parsed

for file in /tmp/knowledge_bank/*.txt; do
  if [ "$file" = "/tmp/knowledge_bank/metadata.txt" ]; then
    continue
  fi
  
  FILE_ID=$(basename "$file" .txt)
  
  # Read full content
  CONTENT=$(cat "$file")
  
  # Parse metadata from content (filename pattern: "Meeting Type - YYYY-MM-DD")
  METADATA=$(grep "$FILE_ID" /tmp/knowledge_bank/metadata.txt)
  FILE_NAME=$(echo "$METADATA" | cut -d'|' -f2)
  MODIFIED=$(echo "$METADATA" | cut -d'|' -f3)
  
  # Extract meeting date from filename if present
  MEETING_DATE=$(echo "$FILE_NAME" | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}' || echo "unknown")
  
  # Create structured JSON entry
  cat > "/tmp/knowledge_bank/parsed/${FILE_ID}.json" <<EOF
{
  "file_id": "$FILE_ID",
  "filename": "$FILE_NAME",
  "meeting_date": "$MEETING_DATE",
  "modified_time": "$MODIFIED",
  "content": $(echo "$CONTENT" | jq -Rs .),
  "word_count": $(echo "$CONTENT" | wc -w),
  "indexed_at": "$(date --iso-8601=seconds)"
}
EOF

done

echo "✓ Parsed and structured all meeting notes"
```

### Step 5: Create Search Index

Build a searchable index with keywords:

```bash
# Create index file
echo "# Meeting Notes Knowledge Bank Index" > /tmp/knowledge_bank/INDEX.md
echo "Generated: $(date)" >> /tmp/knowledge_bank/INDEX.md
echo "" >> /tmp/knowledge_bank/INDEX.md

# List all meetings
echo "## Meetings Indexed" >> /tmp/knowledge_bank/INDEX.md
echo "" >> /tmp/knowledge_bank/INDEX.md

for json_file in /tmp/knowledge_bank/parsed/*.json; do
  FILE_NAME=$(jq -r '.filename' "$json_file")
  MEETING_DATE=$(jq -r '.meeting_date' "$json_file")
  WORD_COUNT=$(jq -r '.word_count' "$json_file")
  FILE_ID=$(jq -r '.file_id' "$json_file")
  
  echo "- **$FILE_NAME** ($MEETING_DATE) - $WORD_COUNT words - ID: $FILE_ID" >> /tmp/knowledge_bank/INDEX.md
done

echo "" >> /tmp/knowledge_bank/INDEX.md
echo "Total meetings: $(ls /tmp/knowledge_bank/parsed/*.json | wc -l)" >> /tmp/knowledge_bank/INDEX.md

cat /tmp/knowledge_bank/INDEX.md
```

### Step 6: Confirm Ingestion Complete

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✓ KNOWLEDGE BANK READY"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Location: /tmp/knowledge_bank/"
echo "Meetings indexed: $(ls /tmp/knowledge_bank/parsed/*.json | wc -l)"
echo "Ready to answer questions!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

Tell the user: "I've ingested and indexed your meeting notes. You can now ask me questions about them!"

---

## PHASE 2: QUERYING

### When User Asks a Question

**Always follow this workflow:**

1. **Check if knowledge bank exists**
2. **Search the knowledge bank for relevant meetings**
3. **Read full content of relevant meetings**
4. **Extract answer grounded in the actual text**
5. **Cite sources with meeting name and date**

### Step 1: Verify Knowledge Bank Exists

```bash
if [ ! -d "/tmp/knowledge_bank/parsed" ]; then
  echo "ERROR: Knowledge bank not found. Please run ingestion first."
  exit 1
fi

if [ $(ls /tmp/knowledge_bank/parsed/*.json 2>/dev/null | wc -l) -eq 0 ]; then
  echo "ERROR: Knowledge bank is empty. Please ingest meeting notes first."
  exit 1
fi
```

### Step 2: Search Knowledge Bank

**Search by keyword in content:**

```bash
# User asks: "What did we decide about API redesign?"
KEYWORD="API redesign"

# Search all parsed notes
grep -l "$KEYWORD" /tmp/knowledge_bank/*.txt 2>/dev/null | while read file; do
  FILE_ID=$(basename "$file" .txt)
  if [ -f "/tmp/knowledge_bank/parsed/${FILE_ID}.json" ]; then
    FILE_NAME=$(jq -r '.filename' "/tmp/knowledge_bank/parsed/${FILE_ID}.json")
    MEETING_DATE=$(jq -r '.meeting_date' "/tmp/knowledge_bank/parsed/${FILE_ID}.json")
    echo "Found in: $FILE_NAME ($MEETING_DATE) - ID: $FILE_ID"
  fi
done
```

**Search by meeting date:**

```bash
# User asks: "What happened in the March 1st meeting?"
TARGET_DATE="2026-03-01"

for json_file in /tmp/knowledge_bank/parsed/*.json; do
  MEETING_DATE=$(jq -r '.meeting_date' "$json_file")
  if [ "$MEETING_DATE" = "$TARGET_DATE" ]; then
    FILE_NAME=$(jq -r '.filename' "$json_file")
    FILE_ID=$(jq -r '.file_id' "$json_file")
    echo "Found: $FILE_NAME - ID: $FILE_ID"
  fi
done
```

**Search by meeting type:**

```bash
# User asks: "Show me all engineering syncs"
MEETING_TYPE="Engineering Sync"

for json_file in /tmp/knowledge_bank/parsed/*.json; do
  FILE_NAME=$(jq -r '.filename' "$json_file")
  if echo "$FILE_NAME" | grep -q "$MEETING_TYPE"; then
    MEETING_DATE=$(jq -r '.meeting_date' "$json_file")
    FILE_ID=$(jq -r '.file_id' "$json_file")
    echo "Found: $FILE_NAME ($MEETING_DATE) - ID: $FILE_ID"
  fi
done
```

### Step 3: Read Full Content

Once you've identified relevant meetings, read their FULL content:

```bash
# Read from parsed JSON
CONTENT=$(jq -r '.content' "/tmp/knowledge_bank/parsed/${FILE_ID}.json")

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Reading: $(jq -r '.filename' "/tmp/knowledge_bank/parsed/${FILE_ID}.json")"
echo "Date: $(jq -r '.meeting_date' "/tmp/knowledge_bank/parsed/${FILE_ID}.json")"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "$CONTENT"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

### Step 4: Extract and Synthesize Answer

**CRITICAL RULES:**
- ✅ Only state what's explicitly in the meeting notes
- ✅ Quote or paraphrase directly from the content
- ✅ Cite which meeting(s) the information came from
- ✅ Include meeting date in citation
- ❌ NEVER make up information
- ❌ NEVER infer beyond what's written
- ❌ NEVER claim something was discussed if it's not in the notes

**Response Format:**

```
**Answer**: [Direct answer from meeting content]

**Source**: [Meeting Name] - [Meeting Date]

**Full Context**: [Relevant excerpt from notes]

**File ID**: [file_id for reference]
```

### Step 5: Handle Multi-Meeting Synthesis

If multiple meetings discuss the topic:

```bash
# Find all relevant meetings
RELEVANT_IDS=()
for file in /tmp/knowledge_bank/*.txt; do
  if grep -q "$KEYWORD" "$file"; then
    FILE_ID=$(basename "$file" .txt)
    RELEVANT_IDS+=("$FILE_ID")
  fi
done

echo "Found ${#RELEVANT_IDS[@]} meetings discussing '$KEYWORD'"

# Read and synthesize
for file_id in "${RELEVANT_IDS[@]}"; do
  FILE_NAME=$(jq -r '.filename' "/tmp/knowledge_bank/parsed/${file_id}.json")
  MEETING_DATE=$(jq -r '.meeting_date' "/tmp/knowledge_bank/parsed/${file_id}.json")
  CONTENT=$(jq -r '.content' "/tmp/knowledge_bank/parsed/${file_id}.json")
  
  echo "═══════════════════════════════════════════════"
  echo "$FILE_NAME - $MEETING_DATE"
  echo "═══════════════════════════════════════════════"
  echo "$CONTENT"
  echo ""
done
```

Then synthesize across all meetings chronologically.

---

## Query Patterns

### Pattern 1: Specific Decision Lookup

```
User: "What was decided about pricing?"

Workflow:
1. Search for "pricing" in all meeting notes
2. Find matching meetings
3. Read full content
4. Extract decision statements
5. Cite source meeting(s)
```

### Pattern 2: Timeline/Progression

```
User: "How did the API discussion evolve?"

Workflow:
1. Search for "API" across all notes
2. Sort matches by meeting date
3. Read each in chronological order
4. Show progression of discussion
5. Cite each meeting in timeline
```

### Pattern 3: Action Items

```
User: "What are my action items?"

Workflow:
1. Search for "action" OR "@" OR "TODO" across notes
2. Extract lines mentioning user's name
3. Group by meeting
4. List with source and date
```

### Pattern 4: Topic Summary

```
User: "Summarize discussions about hiring"

Workflow:
1. Search for "hiring" OR "recruitment" across notes
2. Read all matching meetings
3. Synthesize key points
4. Cite each source
```

---

## Maintenance Operations

### Refresh Knowledge Bank

When new meetings are added:

```bash
# Re-run ingestion to pull new notes
# It will overwrite existing files with same ID
# New files will be added
```

### Check Knowledge Bank Status

```bash
echo "Knowledge Bank Status:"
echo "Location: /tmp/knowledge_bank/"
echo "Total meetings: $(ls /tmp/knowledge_bank/parsed/*.json 2>/dev/null | wc -l)"
echo "Last updated: $(stat -c %y /tmp/knowledge_bank/INDEX.md 2>/dev/null)"
echo ""
echo "Recent meetings:"
ls -t /tmp/knowledge_bank/parsed/*.json | head -5 | while read file; do
  jq -r '"\(.filename) - \(.meeting_date)"' "$file"
done
```

### Search Statistics

```bash
# Show most common keywords
cat /tmp/knowledge_bank/*.txt | tr '[:space:]' '\n' | \
  tr '[:upper:]' '[:lower:]' | \
  grep -v '^$' | \
  sort | uniq -c | sort -rn | head -20
```

---

## Error Handling

### No Knowledge Bank

```bash
if [ ! -d "/tmp/knowledge_bank" ]; then
  echo "⚠ Knowledge bank not initialized."
  echo "Please run: 'ingest my meeting notes from [folder]'"
  exit 1
fi
```

### No Matches Found

```bash
if [ ${#RELEVANT_IDS[@]} -eq 0 ]; then
  echo "No meetings found discussing '$KEYWORD'"
  echo ""
  echo "Try:"
  echo "  - Alternative keywords"
  echo "  - Broader search terms"
  echo "  - Checking meeting date range"
  echo ""
  echo "Available meetings:"
  cat /tmp/knowledge_bank/INDEX.md | grep "^- "
fi
```

### Partial Information

```
"I found partial information about X in the [date] meeting, but the 
discussion appears incomplete. The notes mention [what's there] but 
don't include [what's missing]."
```

---

## Quality Checklist

Before answering any question:

- [ ] Knowledge bank exists and is loaded
- [ ] Searched knowledge bank (not just one file)
- [ ] Read FULL content of relevant meetings
- [ ] Answer is grounded in actual text
- [ ] Cited source meeting(s) with dates
- [ ] Did NOT make up or infer information
- [ ] Acknowledged any gaps or limitations

---

## Example Complete Workflow

```bash
# ═══════════════════════════════════════════════════
# PHASE 1: INGESTION
# ═══════════════════════════════════════════════════

User: "Ingest my meeting notes from the 'Team Meetings' folder"

Claude:
$ gws drive files search --query "name = 'Team Meetings' and mimeType = 'application/vnd.google-apps.folder'" --format json
# Found folder ID: abc123xyz

$ gws drive files list --parent-id "abc123xyz" --query "mimeType = 'application/vnd.google-apps.document'" --format json > /tmp/meeting_files_list.json
# Found 23 meeting notes

$ mkdir -p /tmp/knowledge_bank/parsed
# Exporting all 23 documents...
# [Shows progress]

✓ Knowledge bank built with 23 meeting notes
✓ Ready to answer questions!

# ═══════════════════════════════════════════════════
# PHASE 2: QUERYING
# ═══════════════════════════════════════════════════

User: "What was decided about the Q2 roadmap?"

Claude:
$ grep -l "Q2 roadmap" /tmp/knowledge_bank/*.txt
# Found in 3 meetings

$ # Reading Engineering Sync - 2026-02-15
$ jq -r '.content' /tmp/knowledge_bank/parsed/file123.json

$ # Reading Product Planning - 2026-02-20  
$ jq -r '.content' /tmp/knowledge_bank/parsed/file456.json

$ # Reading All Hands - 2026-03-01
$ jq -r '.content' /tmp/knowledge_bank/parsed/file789.json

**Answer**: The Q2 roadmap was finalized with three priorities:
1. Mobile app redesign (priority #1)
2. API v2 launch (priority #2)  
3. Analytics dashboard (moved to Q3)

**Sources**: 
- Engineering Sync - February 15, 2026
- Product Planning - February 20, 2026
- All Hands - March 1, 2026

**Timeline**:
- Feb 15: Initial proposal discussed
- Feb 20: Priorities debated, mobile chosen as #1
- Mar 1: Final roadmap announced to company

**Key Decision**: Analytics dashboard deprioritized due to 
resource constraints (noted in Feb 20 meeting).
```

---

## Storage Location

Knowledge bank stored at: `/tmp/knowledge_bank/`

Structure:
```
/tmp/knowledge_bank/
├── metadata.txt              # File ID to name mapping
├── INDEX.md                  # Human-readable index
├── *.txt                     # Raw exported content
└── parsed/
    └── *.json               # Structured parsed notes
```

**Note**: This is in `/tmp/` for session storage. For persistent storage across sessions, use a different location or implement saving/loading.

---

## Remember

🎯 **Core Principle**: This skill is a two-phase system:
1. **Ingest once** → Build knowledge bank
2. **Query many times** → Answer from stored knowledge

The knowledge bank is your source of truth. Always read from it, never make things up.
