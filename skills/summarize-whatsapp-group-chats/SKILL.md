---
name: summarize-whatsapp-group-chats
description: Use this skill whenever the user wants to summarize, digest, recap, or get a briefing from WhatsApp group chats using the `wacli` CLI. Trigger on any mention of WhatsApp summaries, group chat digests, catching up on WhatsApp, or briefing from chats. Also trigger when user wants to export or save WhatsApp chat history. Always use `wacli` — no exceptions, never use other methods.
license: MIT
metadata:
  author: Agently
  version: 1.0.1
---

# Summarize WhatsApp Group Chats

Use the `wacli` CLI to fetch and summarize WhatsApp group chats. No exceptions — always use `wacli`.

## Prerequisites

- `wacli` is already installed and authenticated on the user's machine
- If `wacli` commands fail with auth errors, ask the user to re-authenticate before proceeding

---

## Step 1: Discover Available Group Chats

Before fetching messages, list available group chats to find the correct chat name or ID:

```bash
wacli groups list
```

If the user has specified a chat name, fuzzy-match it against the output. Confirm with the user if ambiguous, especially with similar group names.

---

## Step 2: Fetch Messages

Default behavior: fetch the **past 24 hours**. Adjust if the user specifies a different time range.

```bash
# Fetch last 24 hours (default) (please update YYYY-MM-DD below for --after and --before in RFC3339 format)
wacli messages list --chat "<chat-id>" --after "<YYYY-MM-DD HH:MM:SS of one day before today's date>" --before "<YYYY-MM-DD HH:MM:SS of today's date>"

# Fetch by specific date range (if user requests) in RFC3339 format
wacli messages list --chat "<chat-id>" --after "YYYY-MM-DD HH:MM:SS" --before "YYYY-MM-DD HH:MM:SS"

# Fetch by message count (if user requests)
wacli messages list --chat "<chat-id>" --limit 200
```

Capture the raw output. Note the **timestamp of the oldest and newest message** in the fetched data — this becomes the data freshness marker.

---

## Step 3: Export (Optional but Default)

By default, export the raw chat to a `.json` file before summarizing. Filename format:

```
[chat-name]-YYYYMMDD.txt
```

- Use today's date
- Sanitize the chat name: lowercase, replace spaces with hyphens, strip special characters
- Example: `product-team-20250315.txt`

```bash
wacli messages --chat "<chat-id>" --after "<YYYY-MM-DD of one day before today's date>" --before "<YYYY-MM-DD of today's date>" > "[chat-name]-YYYYMMDD.txt"
```

If the user specifies a different filename or path, use that instead.

If user prefers JSON, follow that by adding global `--json` flag the above `wacli` command.

---

## Step 4: Summarize — Chief of Staff Style

Summarize the fetched messages. Write like a **chief of staff briefing an executive**: concise, structured, action-oriented. No padding, no filler.

### Summary Format

```
## [Chat Name] — Summary
**Data covers:** [oldest message timestamp] → [newest message timestamp]
**Fetched at:** [current timestamp]

### Key Decisions
- ...

### Action Items
- [Person] — [what they need to do]

### Notable Discussions
- ...

### FYI / Context
- ...
```

Only include sections that have content. Skip empty sections entirely.

### Summarization Rules

- **Never hallucinate.** Every point must come directly from the fetched messages.
- **Do not invent names, facts, decisions, or action items.** If something is unclear, say so.
- **Be brief.** Aim for signal density — one tight sentence per point.
- **Attribute where it matters.** For decisions and action items, note who said or owns what.
- **Flag urgency.** If something is time-sensitive or unresolved, note it explicitly.
- **Ignore noise.** Skip greetings, reactions, one-word replies, and off-topic banter unless they reveal something important.

---

## Step 5: Handle User Overrides

The user may ask for changes. Always comply. Common overrides:

| User Request | What to Do |
|---|---|
| Different time range | Re-fetch with `--since` / `--until` adjusted |
| Different language | Summarize in the requested language |
| More detail on a topic | Expand that section, pull specific messages |
| Different filename | Rename the export file |
| Skip export | Don't write the `.txt` file |
| Longer / shorter summary | Adjust verbosity accordingly |
| Specific people only | Filter messages by sender in summary |

Whatever the user asks — as long as it's achievable via `wacli` and summarization — do it.

---

## Error Handling

| Error | Response |
|---|---|
| `wacli` not found | Tell user to install and authenticate `wacli` first |
| Auth failure | Ask user to re-run `wacli auth` or equivalent |
| Chat not found | List available chats with `wacli chats` and ask user to confirm |
| No messages in range | Say so clearly; offer to widen the time range |
| Empty output | Do not summarize. Report that no messages were found. |

---

## Quick Reference: wacli Commands

```
Usage:
  wacli messages [command]

Available Commands:
  context     Show message context around a message ID
  list        List messages
  search      Search messages (FTS5 if available; otherwise LIKE)
  show        Show one message

Flags:
  -h, --help   help for messages

Global Flags:
      --json               output JSON instead of human-readable text
      --store string       store directory (default: ~/.wacli)
      --timeout duration   command timeout (non-sync commands) (default 5m0s)
```

```bash
wacli chats                          # List all chats
wacli messages list --chat "<chat-id>" --after "<YYYY-MM-DD of one day before today's date>" --before "<YYYY-MM-DD of today's date>" # Fetch last 24 hours (default) (please update YYYY-MM-DD below for --after and --before)
```

> If `wacli` has different flags on the user's version, run `wacli --help` or `wacli messages --help` to check and adapt.
