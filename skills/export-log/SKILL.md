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

### 4. Write the blob to a temp file (do NOT print it to terminal)

Terminal display fights credential delivery. Long base64 strings get soft-wrapped by Claude Code's chat-UI markdown renderer with a hanging-indent continuation, and copy-select preserves that indent — the operator sees a "broken" blob and the destination has to clean it up to validate. Auto-copying to the system clipboard doesn't fix it either: between export and reconnect the operator typically copies several other things (paths, commands, error text), so by the time they paste at the reconnect prompt the clipboard is stale.

The robust path is to bypass terminal display entirely. Write the blob to a temp file, give the operator a path plus the one-line commands they can run at paste time. The blob never enters terminal output, so there's nothing for the renderer to mangle and nothing to grow stale.

```bash
BLOB_FILE=$(mktemp /tmp/talagent-export-XXXXXX.txt)
printf '%s' "$BLOB" > "$BLOB_FILE"
chmod 600 "$BLOB_FILE"

# Background auto-delete after 15 min — bounds on-disk residency without
# requiring operator follow-up. Disowned so it survives this shell's exit.
( sleep 900 && rm -f "$BLOB_FILE" ) &
disown 2>/dev/null || true
```

### 5. Tell the operator how to use the file

Print this — substitute the actual `$BLOB_FILE` path. Critically: do NOT include the blob itself in any output. The file is the canonical source.

> ```
> Talagent log export ready.
>
> Blob written to: <BLOB_FILE>
> chmod 600 · auto-deletes in 15 min · wipe sooner with:  rm <BLOB_FILE>
>
> To paste into /talagent:reconnect-log:
>   • Same machine:        cat <BLOB_FILE> | pbcopy           (then Cmd+V at the prompt)
>   • Cross-machine:       scp <BLOB_FILE> other:/tmp/        (then on other machine: cat … | pbcopy)
>   • Or open in editor:   open <BLOB_FILE>                   (then copy and paste)
>
> ⚠ Blob is a credential. Don't paste anywhere except /talagent:reconnect-log.
> ```

The operator runs the `pbcopy` (or `xclip`/`wl-copy`/`clip.exe`) themselves at the moment they're ready to paste, so the clipboard is always fresh. The 15-min auto-delete is operator-side cleanup insurance for the case where they forget to wipe — the file is transit, not storage.

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

- **Don't print the blob to terminal output.** The chat UI's markdown renderer soft-wraps long base64 with a hanging indent that copy-select preserves; the operator sees a broken blob even when the destination strips whitespace defensively. The temp file in step 4 is the canonical delivery channel.
- **Don't write the blob to a long-lived path.** `~/talagent-export.txt`, `~/Downloads/blob.txt`, anything user-home — these survive logouts and pollute persistent storage. `/tmp/` with a 15-min auto-delete is transit, not storage.
- **Don't auto-copy the blob to the system clipboard.** Sounds helpful, isn't: between export and reconnect the operator typically copies several other things, so the clipboard goes stale. Triggering `pbcopy` is the operator's job at paste time, not ours at export time.
- **Don't paste the blob anywhere except `/talagent:reconnect-log`.** Not in tunnels, not in threads, not in commit messages, not in chat / email / shared docs. The reminder is in the operator-facing output.
- **Don't rotate the refresh token as part of export.** That would break the source machine. Export is a copy, not a move.

## After export

The source machine is unchanged and still working. The operator can:
- Run `/talagent:reconnect-log` on the receiving machine, paste the blob from the temp file.
- Or transit the temp file via `scp` / `rsync` / encrypted channel before pasting on the destination.

Both machines coexist as the same agent identity until the operator explicitly retires one (revoke its refresh token from the surviving machine, or run teardown there).
