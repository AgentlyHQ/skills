---
name: google-drive-meeting-transcriber
description: Process NEW meeting folders with transcript files and generate executive meeting notes in Google Docs. Use when the user wants to transcribe meetings, process meeting folders, create meeting notes from transcripts, or convert raw transcripts into formatted Google Docs. Trigger on phrases like "transcribe my meetings", "process meeting folders", "create meeting notes from transcripts", "convert transcripts to docs", or when user provides a meetings folder to process. Always use this skill when working with a folder structure containing meeting transcripts that need to be formatted into professional meeting notes.
compatibility:
  - Google Workspace CLI (gws) - https://github.com/googleworkspace/cli
  - Authenticated Google Workspace account with Drive and Docs access
license: MIT
metadata:
  author: Agently
  version: 1.1.0
---

# Meeting Notes Transcriber

Transform unstructured transcript text files inside meeting subfolders into clean, executive-quality meeting notes, written as **Google Docs** in the root folder.

Only processes **new** meeting folders — ones that don't already have a corresponding Google Doc in the root folder.

## What This Does

**Input**: Root Drive folder (e.g., "Meetings") containing subfolders, one per meeting
**Output**: One Google Doc per meeting in the root folder, titled `YYYYMMDD [meeting name]`
**Skip**: Meetings with only audio/video (no transcript `.txt`), or meetings already processed (doc exists)

### Example Structure

```
📁 Meetings/ (ROOT_FOLDER_ID)
  ├── 📁 2026-02-26 12.26.41 Amy & Charlie/
  │   └── zoom_transcript.txt
  ├── 📁 2026-03-01 14.30.00 Team Sync/
  │   └── transcript.txt
  └── 📄 20260226 Amy & Charlie.gdoc   ← created by this skill (skip next run)
  └── 📄 20260301 Team Sync.gdoc       ← created by this skill (skip next run)
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
gws drive files list
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
User: "Transcribe meetings in 'Team Meetings', dry run"
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
  - If `false` (default), skip any meeting folder that already has a matching doc in the root folder
  - If `true`, re-generate and overwrite the existing doc content

- **DRY_RUN** (default: `false`)
  - If `true`, do everything except creating/writing Docs
  - Useful for previewing what would be processed

---

## Processing Workflow

### Step 1: Get ROOT_FOLDER_ID

If user provides folder name, search for it, then look for the folder name inside the `name` variable and get the `id`:

```bash
gws drive files list \
  --format json
```

If user provides URL like `https://drive.google.com/drive/folders/ABC123`, extract `ABC123`.

### Step 2: List All Meeting Subfolders

```bash
gws drive files list \
  --query "'1Yh-tq1Jl479y9dPREVXtBk2ZNZ4SMhOO' in parents and mimeType = 'application/vnd.google-apps.folder' and trashed = false" \
  --format json > /tmp/meeting_folders.json

echo "Found meeting folders:"
jq -r '.files[] | "\(.name) (ID: \(.id))"' /tmp/meeting_folders.json
```

### Step 3: Get Existing Docs in Root Folder (for "new only" detection)

Before processing any folder, fetch the list of Google Docs already in the root folder:

```bash
gws drive files list \
  --query "'${ROOT_FOLDER_ID}' in parents and mimeType = 'application/vnd.google-apps.document' and trashed = false" \
  --format json > /tmp/existing_docs.json

# Build a list of existing doc titles (normalized, lowercase for comparison)
EXISTING_TITLES=$(jq -r '.files[].name' /tmp/existing_docs.json | tr '[:upper:]' '[:lower:]')

echo "Existing docs already processed:"
jq -r '.files[].name' /tmp/existing_docs.json
```

### Step 4: Process Each Meeting Folder

For each meeting folder:

#### 4a. Derive Expected Doc Title

```bash
MEETING_FOLDER_NAME="..."

# Extract date (YYYY-MM-DD prefix)
if [[ "$MEETING_FOLDER_NAME" =~ ^([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then
  MEETING_DATE="${BASH_REMATCH[1]}"
  MEETING_DATE_FORMATTED=$(date -j -f "%Y-%m-%d" "$MEETING_DATE" +"%Y%m%d" 2>/dev/null || date -d "$MEETING_DATE" +"%Y%m%d")
else
  MEETING_DATE_FORMATTED=$(date +"%Y%m%d")
  echo "NOTE: No date prefix found in '$MEETING_FOLDER_NAME', using generation date"
fi

# Strip date and optional timestamp prefix to get the meeting name
MEETING_NAME=$(echo "$MEETING_FOLDER_NAME" | sed -E 's/^[0-9]{4}-[0-9]{2}-[0-9]{2} ([0-9]{2}\.[0-9]{2}\.[0-9]{2} )?//')

DOC_TITLE="${MEETING_DATE_FORMATTED} ${MEETING_NAME}"
```

#### 4b. Skip If Already Processed (OVERWRITE=false)

```bash
DOC_TITLE_LOWER=$(echo "$DOC_TITLE" | tr '[:upper:]' '[:lower:]')

if echo "$EXISTING_TITLES" | grep -qF "$DOC_TITLE_LOWER"; then
  if [ "$OVERWRITE" = "true" ]; then
    echo "OVERWRITE: Re-processing $MEETING_FOLDER_NAME"
    # Get existing doc ID for overwrite
    EXISTING_DOC_ID=$(jq -r --arg title "$DOC_TITLE" \
      '.files[] | select(.name == $title) | .id' /tmp/existing_docs.json)
  else
    echo "SKIPPED_ALREADY_PROCESSED: $MEETING_FOLDER_NAME (doc '$DOC_TITLE' exists)"
    continue
  fi
fi
```

#### 4c. List Files in Meeting Folder

```bash
MEETING_FOLDER_ID="..."

gws drive files list \
  --query "'${MEETING_FOLDER_ID}' in parents and trashed = false" \
  --format json > /tmp/meeting_files_${MEETING_FOLDER_ID}.json
```

#### 4d. Select Transcript Files

Filter for `.txt` files only, excluding audio/video:

```bash
TRANSCRIPT_FILES=$(jq -r '.files[] |
  select(.mimeType == "text/plain" or (.name | endswith(".txt"))) |
  select(.mimeType | (startswith("video/") or startswith("audio/")) | not) |
  "\(.id)|\(.name)"' /tmp/meeting_files_${MEETING_FOLDER_ID}.json)

TRANSCRIPT_COUNT=$(echo "$TRANSCRIPT_FILES" | grep -c . || echo 0)

if [ "$TRANSCRIPT_COUNT" -eq 0 ]; then
  echo "SKIPPED_NO_TRANSCRIPT: $MEETING_FOLDER_NAME"
  continue
fi
```

#### 4e. Download and Merge Transcripts

```bash
mkdir -p /tmp/transcripts/${MEETING_FOLDER_ID}

echo "$TRANSCRIPT_FILES" | while IFS='|' read FILE_ID FILE_NAME; do
  echo "Downloading: $FILE_NAME"
  gws drive files export \
    --file-id "$FILE_ID" \
    --mime-type "text/plain" \
    --output-file "/tmp/transcripts/${MEETING_FOLDER_ID}/${FILE_NAME}"
done

# Merge all transcripts in alphabetical order
# If multiple files, separate them clearly
MERGED_TRANSCRIPT=""
FILE_COUNT=0
for file in $(ls /tmp/transcripts/${MEETING_FOLDER_ID}/*.txt 2>/dev/null | sort); do
  FILE_COUNT=$((FILE_COUNT + 1))
  if [ $FILE_COUNT -gt 1 ]; then
    FILENAME=$(basename "$file")
    MERGED_TRANSCRIPT="${MERGED_TRANSCRIPT}\n\n--- CONTINUED: ${FILENAME} ---\n\n"
  fi
  MERGED_TRANSCRIPT="${MERGED_TRANSCRIPT}$(cat "$file")"
done

echo -e "$MERGED_TRANSCRIPT" > /tmp/merged_transcript_${MEETING_FOLDER_ID}.txt

CHAR_COUNT=$(wc -c < /tmp/merged_transcript_${MEETING_FOLDER_ID}.txt)
if [ "$CHAR_COUNT" -lt 100 ]; then
  echo "SKIPPED_EMPTY_TRANSCRIPT: $MEETING_FOLDER_NAME (only $CHAR_COUNT chars)"
  continue
fi
```

#### 4f. Generate Meeting Notes

Read the full merged transcript and generate structured notes. Claude must produce output in **exactly** this format — no additions, no reordering, no section renaming:

```
MMM D, YYYY | <Host> <> <Counterparty> - <Firm/Context>
Attendees: <Name1>  <Name2>  <Name3>

**Meeting Purpose**
<One-line summary of what this meeting is about.>

**Key Takeaways**
- <Key insight, decision, or conclusion — max 5 points>
- ...

**Discussion**
<Topic Name>
- <Key point attributed to speaker if possible>
- <Decisions, concerns, risks raised>

<Another Topic>
- ...

**Next Steps & Action Items**
- **<Owner>**
  - <Verb + task + context> — Due: <d MMM yy, rough quarter, or leave blank>
- Or, none captured explicitly.
```

**Strict rules for Claude:**
- Use title case, not ALL CAPS, for all section headings
- Use bullet points, not numbered lists
- Do NOT invent facts. If uncertain, keep it general or omit
- Extract attendees from speaker labels in transcript (e.g. "John:", "Speaker 1:")
- The header line (`MMM D, YYYY | ...`) uses the meeting date from the folder name, not today's date
- Summarize — do NOT transcribe verbatim
- Always include **Next Steps & Action Items** (even if "none captured explicitly")
- Always include **Discussion** with topics grouped into sub-sections

#### 4g. Create or Overwrite Google Doc

**Important**: Create the doc directly inside the root folder to avoid a separate move step (which can create duplicate entries).

```bash
if [ "$DRY_RUN" = "false" ]; then
  if [ "$OVERWRITE" = "true" ] && [ -n "$EXISTING_DOC_ID" ]; then
    # Overwrite: clear existing doc content and rewrite
    DOC_ID="$EXISTING_DOC_ID"
    echo "Overwriting existing doc: $DOC_TITLE (ID: $DOC_ID)"
    # Clear content first, then write
    gws docs documents batchUpdate --document-id "$DOC_ID" \
      --json '{"requests":[{"deleteContentRange":{"range":{"startIndex":1,"endIndex":99999}}}]}'
  else
    # Create new doc — specify parent folder at creation time to avoid duplicate placement
    DOC_JSON=$(gws docs documents create \
      --json "{\"title\":\"${DOC_TITLE}\"}" \
      --parents "${ROOT_FOLDER_ID}")
    DOC_ID=$(echo "$DOC_JSON" | jq -r '.documentId')
    echo "Created doc: $DOC_TITLE (ID: $DOC_ID)"
  fi

  # Write meeting notes content to the doc
  gws docs documents batchUpdate --document-id "$DOC_ID" \
    --json "{\"requests\":[{\"insertText\":{\"location\":{\"index\":1},\"text\":$(echo "$NOTES_CONTENT" | jq -Rs .)}}]}"

  echo "PROCESSED: $MEETING_FOLDER_NAME → $DOC_TITLE (ID: $DOC_ID)"
else
  echo "DRY_RUN: Would process $MEETING_FOLDER_NAME → $DOC_TITLE"
fi
```

> **Note on `--parents` flag**: If your version of `gws` does not support `--parents` on `docs documents create`, fall back to creating the doc then immediately moving it:
> ```bash
> DOC_JSON=$(gws docs documents create --json "{\"title\":\"${DOC_TITLE}\"}")
> DOC_ID=$(echo "$DOC_JSON" | jq -r '.documentId')
> CURRENT_PARENTS=$(gws drive files get --file-id "$DOC_ID" --fields 'parents' | jq -r '.parents | join(",")')
> gws drive files update \
>   --file-id "$DOC_ID" \
>   --add-parents "${ROOT_FOLDER_ID}" \
>   --remove-parents "${CURRENT_PARENTS}"
> ```
> Verify the move succeeded before writing content. If the file still appears in the original location, it means `removeParents` did not execute correctly — retry the update.

### Step 5: Summary Report

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TRANSCRIPTION COMPLETE"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Total folders scanned:         $TOTAL_FOLDERS"
echo "Already processed (skipped):   $SKIPPED_EXISTING_COUNT"
echo "Processed (new):               $PROCESSED_COUNT"
echo "Skipped (no transcript):       $SKIPPED_NO_TRANSCRIPT_COUNT"
echo "Skipped (empty transcript):    $SKIPPED_EMPTY_COUNT"
echo "Errors:                        $ERROR_COUNT"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

---

## Skip Rules

### Files to Ignore (Not Transcripts)

**Video formats:** `.mp4`, `.mov`, `.mkv`, `.webm`, `.avi` — or `mimeType` starting with `video/`
**Audio formats:** `.m4a`, `.mp3`, `.wav`, `.aac`, `.flac`, `.ogg` — or `mimeType` starting with `audio/`

### When to Skip a Meeting Folder

- **SKIPPED_ALREADY_PROCESSED**: A Google Doc with the expected title already exists in the root folder (and OVERWRITE=false)
- **SKIPPED_NO_TRANSCRIPT**: No `.txt` files found in the meeting subfolder
- **SKIPPED_EMPTY_TRANSCRIPT**: `.txt` found but total content < 100 characters

---

## Title & Naming Rules

### Date Format: YYYYMMDD

Parse date from folder name:

1. If folder starts with `YYYY-MM-DD`, use that date → convert to `YYYYMMDD`
2. If no date prefix: try to infer from transcript content; fallback to today's date and add note

### Meeting Name

Strip these prefixes from folder name:
- `YYYY-MM-DD `
- `YYYY-MM-DD HH.MM.SS `
- `YYYY-MM-DD HH:MM:SS `

Remaining text becomes the meeting name.

### Doc Title Format

`{YYYYMMDD} {meeting name}`

**Examples:**
- Folder: `2026-02-26 12.26.41 Amy & Charlie` → Doc: `20260226 Amy & Charlie`
- Folder: `2026-03-01 14.30.00 Team Sync` → Doc: `20260301 Team Sync`

---

## Meeting Notes Format (Strict)

The generated content must follow this exact layout — one Google Doc per meeting, one output file only:

```
MMM D, YYYY | <Host> <> <Counterparty> - <Firm/Context>
Attendees: <Name1>  <Name2>  <Name3>

**Meeting Purpose**
<One-line summary.>

**Key Takeaways**
- <Point — max 5>
- ...

**Discussion**
<Topic>
- <Who said what, decisions, concerns>

<Another Topic>
- ...

**Next Steps & Action Items**
- **<Owner>**
  - <Verb + task + context> — Due: <d MMM yy or blank>
- Or, none captured explicitly.
```

**One doc per meeting. No separate summary file, no duplicate.**

---

## Error Handling

### Logging Per Folder

Emit exactly one of:
- **PROCESSED** — meeting folder name, `DOC_ID` of created/updated doc
- **SKIPPED_ALREADY_PROCESSED** — folder already has a matching doc
- **SKIPPED_NO_TRANSCRIPT** — folder has no `.txt`
- **SKIPPED_EMPTY_TRANSCRIPT** — `.txt` found but unusable
- **ERROR_GENERATION** — notes generation failed
- **ERROR_DOC_CREATE** — doc creation failed
- **ERROR_DOC_WRITE** — doc writing failed (include `DOC_ID` for cleanup)

### Data Safety

- Never move, rename, or delete files in meeting subfolders
- Never modify original transcript files
- Only create/update docs in root folder
- If any step fails, skip that meeting and continue to next
- Never produce more than one output file per meeting

---

## Complete Example

### Input Structure

```
📁 Meetings/ (ROOT_FOLDER_ID: abc123)
  ├── 📁 2026-02-26 12.26.41 Amy & Charlie/   ← new, no doc yet
  │   ├── zoom_transcript.txt
  │   └── recording.mp4 (ignored)
  └── 📄 20260301 Team Sync.gdoc              ← already processed, skip
```

### Processing Steps

```bash
# 1. List existing docs in root → find "20260301 Team Sync" already exists
# 2. List meeting subfolders → find "2026-02-26 12.26.41 Amy & Charlie"
# 3. Derive expected title → "20260226 Amy & Charlie" → not in existing docs → process it

# 4. List files in subfolder
$ gws drive files list --query "'folder_id' in parents"
# Found: zoom_transcript.txt (text/plain) ✓
# Found: recording.mp4 (ignored — video) ✗

# 5. Download transcript
$ gws drive files export --file-id "transcript_id" --mime-type "text/plain" --output-file "/tmp/transcript.txt"

# 6. Generate notes (Claude analyzes full transcript content)

# 7. Create Google Doc directly in root folder
$ gws docs documents create --json '{"title":"20260226 Amy & Charlie"}' --parents "abc123"
# Returns: {"documentId": "doc456"}

# 8. Write content to doc
$ gws docs documents batchUpdate --document-id "doc456" --json '...'

✓ PROCESSED: 2026-02-26 12.26.41 Amy & Charlie → 20260226 Amy & Charlie (ID: doc456)
✓ SKIPPED_ALREADY_PROCESSED: 20260301 Team Sync (doc exists)
```

### Output Google Doc

**Title**: `20260226 Amy & Charlie`

**Content** (one doc, exactly this format):
```
Feb 26, 2026 | Amy <> Charlie - Initial Discussion
Attendees: Amy  Charlie

**Meeting Purpose**
Initial conversation about potential collaboration on Q2 project.

**Key Takeaways**
- Both parties interested in moving forward with a partnership
- Budget range: $50K–$100K for Q2
- Timeline: decision by mid-March
- Legal teams need to be involved for contract review
- Technical feasibility confirmed

**Discussion**

Product Scope
- Charlie outlined current platform capabilities and integration options
- Amy shared integration requirements; both agreed on a phased MVP approach

Budget & Timeline
- Amy: budget allocated for Q2, prefers April start
- Charlie: team availability aligns with April; milestone-based payment preferred

Technical Considerations
- API integration estimated at 2–3 weeks
- Dedicated staging environment required
- Charlie to provide technical documentation post-call

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

```bash
# List folders/files
gws drive files list --query "<query>" --format json

# Search for files/folders
gws drive files search --query "<query>" --format json

# Get file metadata
gws drive files get --file-id "<id>" --fields 'parents'

# Export file content (transcripts)
gws drive files export --file-id "<id>" --mime-type "text/plain" --output-file "<path>"

# Move file to folder (update parents)
gws drive files update --file-id "<id>" --add-parents "<folder-id>" --remove-parents "<old-parent-id>"

# Create Google Doc (prefer with --parents to avoid duplicate placement)
gws docs documents create --json '{"title":"<title>"}' --parents "<folder-id>"

# Write to Google Doc via batchUpdate
gws docs documents batchUpdate --document-id "<id>" --json '<requests json>'

# Check schema for exact flags for your gws version
gws schema drive.files.list
gws schema docs.documents.create
gws schema docs.documents.batchUpdate
```

---

## Troubleshooting

### "gws command not found"
```bash
npm install -g @googleworkspace/cli
gws auth login
```

### Two docs appearing per meeting
- The `--parents` flag on `docs documents create` may not be supported by your gws version
- Use the fallback move approach (Step 4g) and verify `removeParents` executed correctly
- Check: `gws drive files get --file-id "<doc-id>" --fields 'parents'` — should show only one parent

### "No transcripts found" but files exist
- Files must be `.txt` with `mimeType = text/plain`
- Check with: `gws drive files list --query "'<folder_id>' in parents" --format json | jq '.files[] | {name, mimeType}'`

### All folders show as "already processed"
- This is correct behavior on re-runs — only new folders without matching docs are processed
- Use `OVERWRITE=true` to force re-processing all meetings

### Doc creation works but writing fails
- Check `gws schema docs.documents.batchUpdate` for supported request types
- The `insertText` request requires content starting at index 1 (not 0)

---

## Minimal Required Capabilities

- Drive: list files/folders, get file metadata, export file content, update file parents
- Docs: create document, batchUpdate (insertText, deleteContentRange)

```bash
gws schema drive.files.list
gws schema drive.files.get
gws schema drive.files.export
gws schema drive.files.update
gws schema docs.documents.create
gws schema docs.documents.batchUpdate
```
