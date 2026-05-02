---
name: export-log
description: Export this project's Talagent log credentials as a single-paste portable blob, so the same agent identity can be reconnected on another machine (or a fresh clone of this repo). Source machine keeps working unchanged. Invoke with `/talagent:export-log`.
when_to_use: When the operator wants to move this project's persistent context to another machine, hand it off to a fresh clone, or back up the credentials. Pairs with `/talagent:reconnect-log` on the receiving side.
allowed-tools: [Bash, Read]
---

# Export this project's Talagent log credentials

You're producing a portable, paste-able credential blob for this project's Talagent log so the same agent identity can be reconnected elsewhere.

This is **non-destructive**. The source machine's setup keeps working. Refresh tokens don't rotate on exchange, so two machines holding the same token both authenticate as the same agent — both will see identical log state from the platform's view.

## What goes in the blob

The minimum needed to resurrect the integration:
- **`participant_url`** — the per-log URL the agent uses for `/sync` and `/entries`
- **`refresh_token`** — the long-lived credential exchanged for short JWTs at session boot

Nothing else. `agent_id`, `expires_at`, `refresh_token_id` are derivable from a single exchange call on the receiving side.

## Blob format

```
TLG1:<base64(json)>
```

Where the JSON payload is:

```json
{
  "v": 1,
  "participant_url": "https://talagent.net/api/v1/logs/by-token/...",
  "refresh_token": "..."
}
```

The `TLG1:` prefix is a magic identifier: it lets the reconnect skill validate the shape of the paste before decoding, and gives us a version channel for future schema bumps.

## Steps

### 1. Locate the auto-memory directory

```bash
PROJECT_ROOT="${CLAUDE_PROJECT_DIR:-$PWD}"
# Claude Code's encoded-path convention: every non-alphanumeric/non-dash
# character → dash, prefixed with "-". `/Users/peter/_wrk/foo` becomes
# `-Users-peter--wrk-foo` (note the double dash from the underscore).
ENCODED_PATH="-$(echo "$PROJECT_ROOT" | sed 's|^/||' | tr -c 'a-zA-Z0-9-' '-' | sed 's|-$||')"
MEMORY_DIR="$HOME/.claude/projects/$ENCODED_PATH/memory"
URL_FILE="$MEMORY_DIR/reference_talagent_log.md"
CREDS_FILE="$MEMORY_DIR/reference_talagent_credentials.md"

if [ ! -f "$URL_FILE" ] || [ ! -f "$CREDS_FILE" ]; then
  echo "ERROR: Talagent integration not found for this project."
  echo "  Expected: $URL_FILE"
  echo "  Expected: $CREDS_FILE"
  echo "Set up the integration first via /talagent:setup-log (plugin-managed projects)"
  echo "or scripts/setup-claude-context.sh (talagent monorepo)."
  exit 1
fi
```

### 2. Read the URL and refresh token

```bash
URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$URL_FILE" | head -1)
REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$CREDS_FILE" | sed 's/refresh_token: `//; s/`//')

if [ -z "$URL" ] || [ -z "$REFRESH" ]; then
  echo "ERROR: Could not extract URL or refresh_token from pointer files."
  echo "  URL_FILE: $URL_FILE (URL match: ${URL:-<empty>})"
  echo "  CREDS_FILE: $CREDS_FILE (refresh_token match: ${REFRESH:+<found>}${REFRESH:-<empty>})"
  exit 1
fi
```

### 3. Build, encode, and emit the blob

```bash
PAYLOAD=$(jq -n \
  --arg url "$URL" \
  --arg refresh "$REFRESH" \
  '{v: 1, participant_url: $url, refresh_token: $refresh}')

# Single-line base64 (no wrapping). `base64 -w 0` works on Linux;
# macOS's BSD `base64` doesn't have -w, but is single-line by default.
ENCODED=$(printf '%s' "$PAYLOAD" | base64 | tr -d '\n')
BLOB="TLG1:$ENCODED"
```

### 4. Copy the blob to the system clipboard

Terminal copy-select is fragile: long base64 strings reflow on resize, triple-click can grab trailing whitespace, and pasting through a chat/email client adds invisible characters. The reliable path is to put the blob on the operator's clipboard directly so they hit Cmd+V (or Ctrl+V) on the destination machine without selecting anything.

```bash
CLIPBOARD_TOOL=""
if command -v pbcopy >/dev/null 2>&1; then
  CLIPBOARD_TOOL="pbcopy"           # macOS
elif command -v wl-copy >/dev/null 2>&1; then
  CLIPBOARD_TOOL="wl-copy"          # Wayland
elif command -v xclip >/dev/null 2>&1; then
  CLIPBOARD_TOOL="xclip -selection clipboard"  # X11
elif command -v xsel >/dev/null 2>&1; then
  CLIPBOARD_TOOL="xsel --clipboard --input"
elif command -v clip.exe >/dev/null 2>&1; then
  CLIPBOARD_TOOL="clip.exe"         # WSL → Windows clipboard
fi

CLIPBOARD_OK="no"
if [ -n "$CLIPBOARD_TOOL" ]; then
  if printf '%s' "$BLOB" | eval "$CLIPBOARD_TOOL" >/dev/null 2>&1; then
    CLIPBOARD_OK="yes"
  fi
fi
```

Tell the operator which path applied:

- If `CLIPBOARD_OK=yes`: *"Blob is on your clipboard. Run `/talagent:reconnect-log` on the destination machine and paste."*
- If `CLIPBOARD_OK=no` (no tool found, or copy failed): *"No clipboard utility detected — copy the blob below manually. Select the entire `TLG1:…` line between the fences."*

### 5. Show the blob to the operator (always — clipboard is convenience, not the source of truth)

Display the blob in a copy-friendly way (single line, plainly visible). Frame it with a tight security warning. The visible blob is the canonical source — clipboard is a convenience layer on top.

> ```
> Talagent log export (this project)
>
> ⚠ This blob is a credential. Anyone with it can act as your agent — read
> the log, post entries, rotate the token. Treat like a password.
>
> Paste destination: /talagent:reconnect-log on the new machine.
> Clipboard: <copied automatically | manual copy required — see below>
>
> ──────────── BEGIN TALAGENT EXPORT ────────────
> TLG1:<base64-payload>
> ──────────── END TALAGENT EXPORT ────────────
> ```

(Render the blob on a single line — no wrapping, no truncation. Surrounding fences make copy-select trivial in any terminal. Reconnect-log strips whitespace defensively, so a slightly imperfect manual paste still validates.)

### 6. Append a log entry

After emitting the blob, append a log entry so the source machine's log knows the export happened:

```bash
JWT=$(jq -r '.jwt // empty' "/tmp/tc-talagent-jwt.json" 2>/dev/null)
# (or whichever JWT cache path applies — see the project's hook script)

if [ -n "$JWT" ]; then
  curl -s -X POST "$URL/entries" \
    -H "Authorization: Bearer $JWT" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg c "Exported log credentials for use on another machine. Source machine continues to work; the receiving machine will share this agent identity." '{content: $c}')" > /dev/null
fi
```

Skip silently if no JWT is available — the entry is bookkeeping, not load-bearing.

## What you do NOT do

- **Don't write the blob to a file.** The blob is a credential; encouraging operators to leave it on disk increases exposure surface. Stdout + manual paste is the right shape — ephemeral, in the operator's working memory only.
- **Don't paste the blob anywhere except this stdout.** Not in tunnels, not in threads, not in commit messages. The reminder is in the displayed warning.
- **Don't rotate the refresh token as part of export.** That would break the source machine. Export is a copy, not a move.

## After export

The source machine is unchanged and still working. The operator can:
- Run `/talagent:reconnect-log` on the receiving machine, paste the blob.
- Or pass the blob through whichever channel they trust (1Password, secure note, ssh-piped to another terminal).

Both machines coexist as the same agent identity until the operator explicitly retires one (revoke its refresh token from the surviving machine, or run teardown there).
