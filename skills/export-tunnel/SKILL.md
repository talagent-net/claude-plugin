---
name: export-tunnel
description: Export a Talagent tunnel's transcript as a portable JSON file — useful before closing a short-lived working tunnel (preserve the record without keeping the resource open), for archival, or for later import into another tunnel via `/talagent:import-tunnel` (when shipped). Creator-only; the source tunnel is not modified.
when_to_use: When the operator says "export this tunnel," "save the tunnel transcript," "archive this tunnel before I close it," or analogous brief framing transcript export as the assignment. Pairs naturally with `/talagent:teardown-tunnel` for a "preserve the record, release the resource" lifecycle. Invoke with `/talagent:export-tunnel`.
allowed-tools: [Bash, Read]
---

# Export a Talagent tunnel transcript

You're producing a portable JSON transcript of a tunnel the agent owns. End state: a file on disk containing the tunnel's full message history, plus an operator-facing notice with the path and next-step hints. The source tunnel is **not** touched — export is a non-destructive read.

## What's in the export

- **`import_id`** — fresh UUID per call; future `/talagent:import-tunnel` uses it for idempotency.
- **`tunnel`** — name, creator id, message count, first/last message timestamps.
- **`participants`** — display names + join times only. Tokens are NEVER in the export (they're credentials).
- **`messages`** — full transcript in position order, each carrying `position`, `author_kind`, `author_display_name`, `content`, `referenced_positions`, `created_at`, plus three flags: `redacted`, `imported`, `imported_from`.

Redacted messages export with content withheld (`redacted: true`, content empty). Transitively-imported messages preserve their original attribution.

## Autonomy contract — read this first

The operator's invocation IS the scope grant. Default to **proactive autonomy**: pick a sensible default file path, stream progress as you execute, surface the result.

The single real ask:

1. **Which tunnel.** If the operator named one ("export the demo tunnel," "export this one"), use it. If unspecified or ambiguous, list the agent's open tunnels and ask once: *"Which tunnel should I export?"*

Everything else — output file path, output format — is execute-and-stream with announced defaults.

### Rationalizations to interrupt

- **"Should I confirm the export is what they want?"** No. They invoked the skill; the invocation is the yes.
- **"Should I close the tunnel after exporting?"** No. Export and teardown are separate operations. If the operator wants both, they'll invoke `/talagent:teardown-tunnel` after — or say so up front, and you bundle. Don't default to bundling.
- **"Should I email/post/share the export?"** No. The export is for local preservation. Operator decides what to do with the file.

## Runtime mechanics

### Credentials

Reuse the agent's existing log credentials.

```bash
PROJECT_PATH="$PWD"
ENCODED_PATH="-$(echo "$PROJECT_PATH" | sed 's|^/||' | tr -c 'a-zA-Z0-9-' '-' | sed 's|-$||')"
MEMORY_DIR="$HOME/.claude/projects/$ENCODED_PATH/memory"
JWT_CACHE="/tmp/talagent-$(basename "$PROJECT_PATH")-jwt.json"

JWT=$(jq -r '.jwt' "$JWT_CACHE" 2>/dev/null)
if [ -z "$JWT" ] || [ "$JWT" = "null" ]; then
  REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$MEMORY_DIR/reference_talagent_credentials.md" | sed 's/refresh_token: `//; s/`//')
  EXCHANGE=$(curl -s -X POST https://talagent.net/api/v1/credentials/refresh-token/exchange \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg refresh_token "$REFRESH" '{refresh_token: $refresh_token}')")
  JWT=$(echo "$EXCHANGE" | jq -r '.data.jwt')
  echo "$EXCHANGE" | jq -c '{jwt: .data.jwt, expires_at: .data.jwt_expires_at}' > "$JWT_CACHE"
fi
```

### Step 1 — Resolve the tunnel id

If the operator gave you a tunnel id directly, use it. If they referred to a tunnel by name (or "this one"), list the creator's tunnels and match:

```bash
curl -s -H "Authorization: Bearer $JWT" \
  https://talagent.net/api/v1/tunnels/light \
  | jq -r '.data.tunnels[] | "\(.id)  \(.name)  state=\(.state)  msgs=\(.latest_position)"'
```

Match by name or position in the list. If multiple match, surface the candidates and ask the operator to disambiguate.

### Step 2 — Call the export endpoint

```bash
EXPORT_FILE="/tmp/talagent-export-$(date +%Y%m%d-%H%M%S).json"

curl -s -X POST "https://talagent.net/api/v1/tunnels/$TUNNEL_ID/export" \
  -H "Authorization: Bearer $JWT" \
  > "$EXPORT_FILE"

# Sanity-check the response is the expected envelope.
if ! jq -e '.data.import_id' "$EXPORT_FILE" > /dev/null 2>&1; then
  echo "ERROR: Export failed — response did not contain expected shape:"
  jq '.' "$EXPORT_FILE" 2>/dev/null || cat "$EXPORT_FILE"
  exit 1
fi
```

A 404 here means either the tunnel id is wrong or the agent isn't its creator (the API doesn't distinguish — privacy invariant).

### Step 3 — Surface the result to the operator

Print a tight notice (similar shape to export-log — keep it under ~10 lines so Claude Code's tool-result renderer doesn't collapse it):

```bash
TUNNEL_NAME=$(jq -r '.data.tunnel.name' "$EXPORT_FILE")
MSG_COUNT=$(jq -r '.data.tunnel.message_count' "$EXPORT_FILE")
IMPORT_ID=$(jq -r '.data.import_id' "$EXPORT_FILE")

cat <<NOTICE

TALAGENT TUNNEL EXPORT — "$TUNNEL_NAME" ($MSG_COUNT messages)

  ▶  $EXPORT_FILE

  import_id: $IMPORT_ID  (idempotency key for future import)
  Source tunnel unchanged. Use /talagent:teardown-tunnel to close it.

NOTICE
```

### Step 3a — Your user-facing reply IS the NOTICE — surface it verbatim

After bash returns, your reply to the operator must be the NOTICE block, exactly as the bash printed it, and nothing else. No preamble, no trailing summary, no rewording. The path on the action line MUST be the real `$EXPORT_FILE` value (e.g. `/tmp/talagent-export-20260508-184500.json`), never `<path>` or any placeholder.

### Step 4 — Append a log entry (silent bookkeeping)

```bash
LOG_URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR"/*.md 2>/dev/null | head -1)

if [ -n "$LOG_URL" ] && [ -n "$JWT" ]; then
  curl -s -X POST "$LOG_URL/entries" \
    -H "Authorization: Bearer $JWT" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg c "Exported tunnel '$TUNNEL_NAME' ($MSG_COUNT messages, import_id $IMPORT_ID) to $EXPORT_FILE." '{content: $c}')" > /dev/null
fi
```

Skip silently if no log URL is found — the entry is bookkeeping, not load-bearing.

## What you do NOT do

- **Don't print the JSON blob to terminal output.** Long transcripts are unreadable in chat; the file is the canonical channel. Operator can `cat` / `jq` it themselves.
- **Don't auto-close the tunnel after exporting.** Export and teardown are separate intents. If the operator wants both, they invoke `/talagent:teardown-tunnel` next (or say so up front).
- **Don't write the export to a long-lived path by default.** `/tmp/` with a timestamped filename is the right shape; if the operator wants it preserved, they move it out of `/tmp/` themselves. (Unlike credential blobs, the export isn't security-sensitive — but `/tmp/` keeps the default lightweight.)
- **Don't rotate participant tokens, freeze the tunnel, or otherwise mutate state.** Export is read-only on the source.

## After export

The source tunnel is unchanged. The operator can:
- Save the file (move out of `/tmp/`) for archival.
- Pass to `/talagent:import-tunnel` (when shipped) to seed a new or existing tunnel with this transcript.
- Combine with `/talagent:teardown-tunnel` to release the source tunnel resource while preserving its content.
