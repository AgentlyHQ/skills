---
name: google-drive-meeting-transcriber
description: Process meeting folders with transcript files and generate executive meeting notes in Google Docs. Use when the user wants to transcribe meetings, process meeting folders, create meeting notes from transcripts, or convert raw transcripts into formatted Google Docs. Trigger on phrases like "transcribe my meetings", "process meeting folders", "create meeting notes from transcripts", "convert transcripts to docs", or when user provides a meetings folder to process. Always use this skill when working with a folder structure containing meeting transcripts that need to be formatted into professional meeting notes.
compatibility:
  - Google Workspace CLI (gws) - https://github.com/googleworkspace/cli
  - Authenticated Google Workspace account with Drive and Docs access
license: MIT
metadata:
  author: Agently
  version: 1.0.0
---

# Meeting Notes Transcriber

Transform unstructured transcript text files inside meeting subfolders into clean, executive-quality meeting notes, written as **Google Docs** in the root folder.

## What This Does

**Input**: Root Drive folder (e.g., "Meetings") containing subfolders, one per meeting  
**Output**: One Google Doc per meeting in the root folder, titled `YYYYMMDD [meeting name]`  
**Skip**: Meetings with only audio/video (no transcript `.txt`)

### Example Structure

```
📁 Meetings/ (ROOT_FOLDER_ID)
  ├── 📁 2026-02-26 12.26.41 Amy and Charlie/
  │   └── zoom_transcript.txt
  ├── 📁 2026-03-01 14.30.00 Team Sync/
  │   └── transcript.txt
  └── 📄 20260226 Amy and Charlie.gdoc (created by this skill)
  └── 📄 20260301 Team Sync.gdoc (created by this skill)
```

---

## Prerequisites

```bash
# Check if gws is installed
gws --version

# If not installed
npm install -g @googleworkspace/cli

# Authenticate
gws auth login

# Verify access
gws drive files list --max-results 5
```

---

## Usage

### Basic Command

```
User: "Transcribe meetings in folder [folder name or ID]"
```

### With Options

```
User: "Transcribe meetings in 'Team Meetings' folder, overwrite existing docs"
```

---

## Input Parameters

### Required

- **ROOT_FOLDER_ID**: Drive folder ID for the "Meetings" root folder
  - User can provide folder name (you search for it)
  - User can provide folder ID directly
  - User can provide folder URL (extract ID from it)

### Optional

- **OVERWRITE** (default: `false`)
  - If `true`, and a doc with the same title exists in root folder, overwrite its contents
  - If `false`, create new doc even if same title exists

- **DRY_RUN** (default: `false`)
  - If `true`, do everything except creating/writing Docs
  - Useful for previewing what would be processed

---

## Processing Workflow

### Step 1: Get ROOT_FOLDER_ID

If user provides folder name, search for it:

```bash
gws drive files search \
  --query "name = 'Meetings' and mimeType = 'application/vnd.google-apps.folder'" \
  --format json | jq -r '.files[0].id'
```

If user provides URL like `https://drive.google.com/drive/folders/ABC123`, extract `ABC123`.

### Step 2: List All Meeting Subfolders

```bash
gws drive files list \
  --query "'${ROOT_FOLDER_ID}' in parents and mimeType = 'application/vnd.google-apps.folder' and trashed = false" \
  --format json > /tmp/meeting_folders.json

# Show what we found
echo "Found meeting folders:"
jq -r '.files[] | "\(.name) (ID: \(.id))"' /tmp/meeting_folders.json
```

### Step 3: Process Each Meeting Folder

For each meeting folder:

#### 3a. List Files in Meeting Folder

```bash
MEETING_FOLDER_ID="..."
MEETING_FOLDER_NAME="..."

gws drive files list \
  --query "'${MEETING_FOLDER_ID}' in parents and trashed = false" \
  --format json > /tmp/meeting_files_${MEETING_FOLDER_ID}.json
```

#### 3b. Select Transcript Files

Filter for `.txt` files, excluding audio/video:

```bash
# Get transcript files (text/plain or .txt extension)
TRANSCRIPT_FILES=$(jq -r '.files[] | 
  select(.mimeType == "text/plain" or (.name | endswith(".txt"))) | 
  select(.mimeType | startswith("video/") or startswith("audio/") | not) |
  "\(.id)|\(.name)"' /tmp/meeting_files_${MEETING_FOLDER_ID}.json)

# Count transcripts
TRANSCRIPT_COUNT=$(echo "$TRANSCRIPT_FILES" | grep -c . || echo 0)

if [ "$TRANSCRIPT_COUNT" -eq 0 ]; then
  echo "SKIPPED_NO_TRANSCRIPT: $MEETING_FOLDER_NAME"
  continue
fi
```

#### 3c. Download and Merge Transcripts

```bash
# Create temp directory for this meeting
mkdir -p /tmp/transcripts/${MEETING_FOLDER_ID}

# Download each transcript file
echo "$TRANSCRIPT_FILES" | while IFS='|' read FILE_ID FILE_NAME; do
  echo "Downloading: $FILE_NAME"
  gws drive files export \
    --file-id "$FILE_ID" \
    --mime-type "text/plain" \
    --output-file "/tmp/transcripts/${MEETING_FOLDER_ID}/${FILE_NAME}"
done

# Merge all transcripts in alphabetical order
MERGED_TRANSCRIPT=""
for file in $(ls /tmp/transcripts/${MEETING_FOLDER_ID}/*.txt | sort); do
  FILENAME=$(basename "$file")
  MERGED_TRANSCRIPT="${MERGED_TRANSCRIPT}\n\n--- FILE: ${FILENAME} ---\n\n"
  MERGED_TRANSCRIPT="${MERGED_TRANSCRIPT}$(cat "$file")"
done

# Save merged transcript
echo -e "$MERGED_TRANSCRIPT" > /tmp/merged_transcript_${MEETING_FOLDER_ID}.txt

# Check if transcript is usable (at least 100 characters)
CHAR_COUNT=$(wc -c < /tmp/merged_transcript_${MEETING_FOLDER_ID}.txt)
if [ "$CHAR_COUNT" -lt 100 ]; then
  echo "SKIPPED_EMPTY_TRANSCRIPT: $MEETING_FOLDER_NAME (only $CHAR_COUNT chars)"
  continue
fi
```

#### 3d. Parse Meeting Metadata

Extract date and meeting name from folder name:

```bash
# Meeting folder name format: "YYYY-MM-DD HH.MM.SS Meeting Name"
# or "YYYY-MM-DD Meeting Name"

# Extract date (YYYY-MM-DD prefix)
if [[ "$MEETING_FOLDER_NAME" =~ ^([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then
  MEETING_DATE="${BASH_REMATCH[1]}"
  MEETING_DATE_FORMATTED=$(date -d "$MEETING_DATE" +"%Y%m%d")
  MEETING_DATE_DISPLAY=$(date -d "$MEETING_DATE" +"%b %d, %Y")
else
  # Fallback to today's date
  MEETING_DATE_FORMATTED=$(date +"%Y%m%d")
  MEETING_DATE_DISPLAY=$(date +"%b %d, %Y")
  echo "NOTE: No date prefix found, using generation date"
fi

# Extract meeting name (everything after date and timestamp)
MEETING_NAME=$(echo "$MEETING_FOLDER_NAME" | sed -E 's/^[0-9]{4}-[0-9]{2}-[0-9]{2} ([0-9]{2}\.[0-9]{2}\.[0-9]{2} )?//')

# Create doc title
DOC_TITLE="${MEETING_DATE_FORMATTED} ${MEETING_NAME}"
```

#### 3e. Generate Meeting Notes

Read the merged transcript and generate structured notes:

```bash
# Read transcript
TRANSCRIPT_CONTENT=$(cat /tmp/merged_transcript_${MEETING_FOLDER_ID}.txt)

# Use Claude to generate meeting notes from transcript
# This is where Claude analyzes the transcript and creates structured notes
```

**Meeting Notes Template** (to be populated by Claude):

```markdown
MMM D, YYYY | <Founder/Host> <> <Counterparty> - <Firm/Context>
Attendees: <Name1>  <Name2>  <Name3>

**Meeting Purpose**
Short 1-liner on what this meeting is about.

**Key Takeaways**
- (Please summarise ALL key takeaways here, max 5 points)
- ...

**Discussion** (If you can, try to categorise them into topics)
- Capture key discussions here, especially who said what
- Risks, concerns, decisions made
- Key questions asked

**Next Steps & Action Items**
- **<Owner>**
  - <Verb + task + context> — Due: <d MMM yy or blank or rough quarter of year>
- Or, none captured explicitly.
```

**Style Guidelines for Claude:**
- Crisp, factual, skimmable. No fluff.
- Summarize, don't just "clean up" the transcript
- Do NOT invent facts. If uncertain, keep it general
- Extract attendees from speaker labels in transcript
- Identify firms/contexts from capitalized entities
- Always include **Next Steps & Action Items** section (even if "none captured")

#### 3f. Check for Existing Doc (OVERWRITE logic)

```bash
if [ "$OVERWRITE" = "true" ]; then
  # Search for existing doc with same title in root folder
  EXISTING_DOC=$(gws drive files search \
    --query "name = '${DOC_TITLE}' and '${ROOT_FOLDER_ID}' in parents and mimeType = 'application/vnd.google-apps.document' and trashed = false" \
    --format json | jq -r '.files[0].id // empty')
  
  if [ -n "$EXISTING_DOC" ]; then
    echo "Found existing doc, will overwrite: $DOC_TITLE (ID: $EXISTING_DOC)"
    DOC_ID="$EXISTING_DOC"
    OVERWRITE_MODE=true
  else
    OVERWRITE_MODE=false
  fi
else
  OVERWRITE_MODE=false
fi
```

#### 3g. Create Google Doc (if not overwriting)

```bash
if [ "$OVERWRITE_MODE" = "false" ] && [ "$DRY_RUN" = "false" ]; then
  # Create new Google Doc
  DOC_JSON=$(gws docs documents create --json "{\"title\":\"${DOC_TITLE}\"}")
  DOC_ID=$(echo "$DOC_JSON" | jq -r '.documentId')
  
  echo "Created doc: $DOC_TITLE (ID: $DOC_ID)"
  
  # Move doc to root Meetings folder
  # Get current parents
  PARENTS_JSON=$(gws drive files get --file-id "$DOC_ID" --fields 'parents')
  REMOVE_PARENTS=$(echo "$PARENTS_JSON" | jq -r '.parents | join(",")')
  
  # Add to root folder, remove from current location
  gws drive files update \
    --file-id "$DOC_ID" \
    --params "{\"addParents\": \"${ROOT_FOLDER_ID}\", \"removeParents\": \"${REMOVE_PARENTS}\"}"
  
  echo "Moved doc to root folder"
fi
```

#### 3h. Write Content to Google Doc

```bash
if [ "$DRY_RUN" = "false" ]; then
  # Write the meeting notes to the doc
  # Note: gws docs +write syntax may vary by version
  # Check with: gws schema docs.documents.batchUpdate
  
  gws docs +write --document-id "$DOC_ID" --text "$NOTES_MARKDOWN"
  
  echo "✓ PROCESSED: $MEETING_FOLDER_NAME → $DOC_TITLE (ID: $DOC_ID)"
else
  echo "✓ DRY_RUN: Would process $MEETING_FOLDER_NAME → $DOC_TITLE"
fi
```

### Step 4: Summary Report

At the end, show summary:

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TRANSCRIPTION COMPLETE"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Total folders scanned: $TOTAL_FOLDERS"
echo "Processed: $PROCESSED_COUNT"
echo "Skipped (no transcript): $SKIPPED_NO_TRANSCRIPT_COUNT"
echo "Skipped (empty transcript): $SKIPPED_EMPTY_COUNT"
echo "Errors: $ERROR_COUNT"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

---

## Skip Rules

### Files to Ignore (Not Transcripts)

**Video formats:**
- `.mp4`, `.mov`, `.mkv`, `.webm`, `.avi`
- Any file with `mimeType` starting with `video/`

**Audio formats:**
- `.m4a`, `.mp3`, `.wav`, `.aac`, `.flac`, `.ogg`
- Any file with `mimeType` starting with `audio/`

### When to Skip a Meeting Folder

- **SKIPPED_NO_TRANSCRIPT**: No `.txt` files found
- **SKIPPED_EMPTY_TRANSCRIPT**: Transcript found but < 100 characters (unusable)

---

## Title & Naming Rules

### Date Format: YYYYMMDD

Parse date from folder name:

1. If folder starts with `YYYY-MM-DD`, use that date
2. Convert to `YYYYMMDD` (remove dashes)
3. If no `YYYY-MM-DD` prefix:
   - Try to infer from transcript content (best-effort)
   - Fallback to today's date (last resort)
   - Add note: "This meeting has no inferred title, defaulting to the date of generation."

### Meeting Name

Strip these prefixes from folder name:
- `YYYY-MM-DD `
- `YYYY-MM-DD HH.MM.SS `
- `YYYY-MM-DD HH:MM:SS `

Remaining text becomes the meeting name.

### Doc Title Format

`{YYYYMMDD} {meeting name}`

**Examples:**
- Folder: `2026-02-26 12.26.41 Amy and Charlie`
- Doc Title: `20260226 Amy and Charlie`

---

## Error Handling

### Logging Per Folder

Emit exactly one of:
- **PROCESSED** — include meeting folder name, created/updated `DOC_ID`
- **SKIPPED_NO_TRANSCRIPT** — folder has no `.txt`
- **SKIPPED_EMPTY_TRANSCRIPT** — `.txt` found but content unusable
- **ERROR_GENERATION** — notes generation failed
- **ERROR_DOC_CREATE** — doc creation failed
- **ERROR_DOC_WRITE** — doc writing failed (include `DOC_ID` for cleanup)

### Data Safety

- ✅ Never move, rename, or delete files in meeting subfolders
- ✅ Never modify original transcript files
- ✅ Only create/update docs in root folder
- ✅ If any step fails, skip that meeting and continue to next

---

## Notes Generation Guidelines

When Claude analyzes transcripts to generate meeting notes:

### 1. Extract Metadata
- **Attendees**: Look for speaker labels (e.g., "Speaker 1:", "John:", etc.)
- **Firms/Context**: Identify from capitalized entities, company mentions
- **Meeting Purpose**: Infer from opening remarks or recurring themes

### 2. Identify Key Takeaways
- Max 5 points
- Focus on decisions, conclusions, important insights
- Not just "what was discussed" but "what was decided/learned"

### 3. Organize Discussion
- Group by topic when possible
- Capture who said what (attribute to speakers)
- Note risks, concerns, decisions
- Include key questions asked

### 4. Extract Action Items
- Look for commitments, tasks, follow-ups
- Format: `**[Owner]**: [Verb] [task] [context] — Due: [date]`
- If no explicit action items, state: "none captured explicitly"

### 5. Style
- Crisp, factual, skimmable
- No fluff or filler
- Summarize, don't transcribe verbatim
- If uncertain about a fact, keep it general
- Use bullet points, bold for emphasis

---

## Complete Example

### Input Structure

```
📁 Meetings/ (ROOT_FOLDER_ID: abc123)
  └── 📁 2026-02-26 12.26.41 Amy and Charlie/
      ├── zoom_transcript.txt
      └── recording.mp4 (ignored)
```

### Processing Steps

```bash
# 1. List meeting folders
$ gws drive files list --query "'abc123' in parents and mimeType = 'application/vnd.google-apps.folder'"

# 2. For folder "2026-02-26 12.26.41 Amy and Charlie"
$ gws drive files list --query "'folder_id' in parents"
# Found: zoom_transcript.txt (text/plain)
# Found: recording.mp4 (ignored - video)

# 3. Download transcript
$ gws drive files export --file-id "transcript_id" --mime-type "text/plain" --output-file "/tmp/transcript.txt"

# 4. Generate notes from transcript
# [Claude analyzes and creates structured notes]

# 5. Create Google Doc
$ gws docs documents create --json '{"title":"20260226 Amy and Charlie"}'
# Returns: {"documentId": "doc123"}

# 6. Move to root folder
$ gws drive files update --file-id "doc123" --params '{"addParents": "abc123", "removeParents": "..."}'

# 7. Write content
$ gws docs +write --document-id "doc123" --text "$NOTES_MARKDOWN"

✓ PROCESSED: 2026-02-26 12.26.41 Amy and Charlie → 20260226 Amy and Charlie (ID: doc123)
```

### Output Google Doc

**Title**: `20260226 Amy and Charlie`

**Content**:
```
Feb 26, 2026 | Amy <> Charlie - Initial Discussion
Attendees: Amy  Charlie

**Meeting Purpose**
Initial conversation about potential collaboration on Q2 project.

**Key Takeaways**
- Both parties interested in moving forward with partnership
- Budget range: $50K-$100K for Q2
- Timeline: Decision by mid-March
- Need to involve legal teams for contract review
- Technical feasibility confirmed

**Discussion**

Product Scope
- Charlie outlined current platform capabilities
- Amy shared requirements for integration
- Both agreed on phased approach starting with MVP

Budget & Timeline
- Amy: Budget allocated for Q2, prefer to start in April
- Charlie: Team availability aligns with April start
- Discussion on payment terms (milestone-based preferred)

Technical Considerations
- API integration will take 2-3 weeks
- Need dedicated staging environment
- Charlie to provide technical documentation

**Next Steps & Action Items**
- **Charlie**
  - Send technical documentation and API specs — Due: 5 Mar 26
  - Provide cost breakdown for phased approach — Due: 8 Mar 26
- **Amy**
  - Review documentation with internal team — Due: 12 Mar 26
  - Schedule legal review meeting — Due: mid-March
  - Confirm Q2 budget allocation — Due: 15 Mar 26
```

---

## CLI Command Reference

### Essential gws Commands

```bash
# List folders
gws drive files list --query "<query>" --format json

# Search for files/folders
gws drive files search --query "<query>" --format json

# Get file metadata
gws drive files get --file-id "<id>" --fields 'parents'

# Export file content
gws drive files export --file-id "<id>" --mime-type "text/plain" --output-file "<path>"

# Update file (move to folder)
gws drive files update --file-id "<id>" --params '{"addParents":"<id>","removeParents":"<id>"}'

# Create Google Doc
gws docs documents create --json '{"title":"<title>"}'

# Write to Google Doc
gws docs +write --document-id "<id>" --text "<content>"

# Check schema for exact flags
gws schema drive.files.list
gws schema docs.documents.create
```

---

## Testing Before Full Run

### Dry Run Mode

```
User: "Transcribe meetings in 'Team Meetings' folder in dry run mode"
```

This will:
- List all meeting folders
- Check for transcripts
- Show what would be processed
- NOT create any docs

### Single Folder Test

Process just one meeting folder first:
```
User: "Transcribe just the '2026-02-26 Amy and Charlie' meeting"
```

---

## Troubleshooting

### "gws command not found"
```bash
npm install -g @googleworkspace/cli
gws auth login
```

### "Permission denied" errors
- Ensure `gws auth login` was successful
- Check that account has access to the Meetings folder
- Verify Docs API is enabled

### "No transcripts found" but files exist
- Check file extensions (must be `.txt`)
- Verify files are not actually audio/video
- Check mimeType is `text/plain`

### Transcript content looks wrong
- Check file encoding (should be UTF-8)
- Verify file isn't corrupted
- Try manually downloading to inspect

### Doc creation works but writing fails
- Check `gws docs +write` syntax for your gws version
- Use `gws schema docs.documents.batchUpdate` to see available methods
- May need to use `batchUpdate` instead of `+write`

---

## Performance Notes

- Processing 10 meetings takes ~2-5 minutes
- Most time spent on:
  - Downloading transcript files
  - Claude analyzing transcripts
  - Creating Google Docs

- Optimization tips:
  - Use dry run first to validate structure
  - Process in batches if many meetings
  - Can parallelize folder processing if needed

---

## Minimal Required Capabilities

This skill requires:
- ✅ Drive: list files/folders
- ✅ Drive: get file metadata
- ✅ Drive: export file content
- ✅ Drive: update file parents (move)
- ✅ Docs: create document
- ✅ Docs: write/update document content

Check availability:
```bash
gws schema drive.files.list
gws schema drive.files.get
gws schema drive.files.export
gws schema drive.files.update
gws schema docs.documents.create
gws schema docs.documents.batchUpdate
```

---

## Remember

🎯 **Core Workflow**:
1. Find meeting folders with transcripts
2. Download and merge transcript files
3. Generate structured notes (Claude analyzes)
4. Create Google Doc in root folder
5. Write formatted notes to doc

The skill transforms raw transcripts into professional meeting notes that are skimmable, actionable, and properly organized.
