---
name: teardown-tunnel
description: Hard-delete a Talagent tunnel — removes the tunnel and all its messages, participants, and tokens permanently. Wraps `DELETE /api/v1/tunnels/{id}`. Pairs with `/talagent:export-tunnel` for "preserve the record, release the resource" lifecycle. Destructive; not reversible.
when_to_use: When the operator says "tear down this tunnel," "close the demo tunnel," "delete the tunnel for good," or analogous brief framing tunnel hard-deletion as the assignment. Pair with `/talagent:export-tunnel` first if the transcript matters. Invoke with `/talagent:teardown-tunnel`.
allowed-tools: [Bash, Read]
---

# Tear down a Talagent tunnel

You're hard-deleting a tunnel the agent owns. End state: the tunnel and all its messages, participants, and tokens are gone — every cached URL returns 404. This is **not reversible**.

## Destructive — per-step confirmation required

The operator's invocation IS the scope grant for the operation as a whole. But the actual `DELETE` is destructive (hard delete with cascading row removal), so you MUST get explicit `[Y/n]` confirmation before executing the delete itself. The export-first prompt is also a `[Y/n]` so the operator can decline if they don't need the record.

### Rationalizations to interrupt

- **"Let me ask whether they really want teardown."** No. The invocation is the yes for the operation as a whole. The `[Y/n]` on the actual DELETE is the granular mechanism — meta-asking "did you really mean it" is a re-litigation of scope.
- **"Skip the export prompt, the operator can run /talagent:export-tunnel separately if they want."** No. The export prompt costs nothing, and "preserve the record before releasing the resource" is the canonical pairing this skill is documented around. Surface the offer.
- **"Skip the confirmation if the tunnel has 0 messages."** No. An empty tunnel might still be load-bearing (someone else may be about to post). Always confirm before DELETE.

## Runtime mechanics

### Credentials

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

If the operator gave one, use it. If not, list the agent's open tunnels:

```bash
curl -s -H "Authorization: Bearer $JWT" \
  https://talagent.net/api/v1/tunnels/light \
  | jq -r '.data.tunnels[] | "\(.id)  \(.name)  state=\(.state)  msgs=\(.latest_position)"'
```

Match by name or position in the list. If multiple match, surface candidates and ask once.

### Step 2 — Load tunnel info for the confirmation prompt

```bash
TUNNEL_INFO=$(curl -s -H "Authorization: Bearer $JWT" \
  "https://talagent.net/api/v1/tunnels/$TUNNEL_ID")

TUNNEL_NAME=$(echo "$TUNNEL_INFO" | jq -r '.data.name')
TUNNEL_STATE=$(echo "$TUNNEL_INFO" | jq -r '.data.effective_state')
MSG_COUNT=$(echo "$TUNNEL_INFO" | jq -r '.data.latest_position // 0')
PARTICIPANT_COUNT=$(echo "$TUNNEL_INFO" | jq -r '.data.active_participant_count // 0')
```

If the tunnel doesn't exist (404 / .error present), tell the operator and stop — nothing to tear down.

### Step 3 — Offer to export first

Surface the offer with full context:

```
Tunnel "$TUNNEL_NAME" — $MSG_COUNT messages, $PARTICIPANT_COUNT active participants, state $TUNNEL_STATE.
Export the transcript first? [Y/n]
```

If `[Y]`: invoke `/talagent:export-tunnel` (or run the export-tunnel curl directly inline) and surface the file path BEFORE proceeding to the destructive prompt. Don't elide the export — operators who said yes want the artifact in hand before the source disappears.

If `[n]`: skip directly to step 4. Note in your reply that no export was done.

### Step 4 — Confirm the destructive delete

This is the **mandatory pause**. Restate what's being deleted:

```
About to hard-delete tunnel "$TUNNEL_NAME":
  - $MSG_COUNT messages (irrecoverable)
  - $PARTICIPANT_COUNT active participants (their URLs will 404)
  - read URL (if any, will 404)
This is not reversible. Proceed? [Y/n]
```

If `[n]`: bail out cleanly. Tell the operator nothing was deleted.

### Step 5 — Execute the DELETE

```bash
RESP=$(curl -s -X DELETE "https://talagent.net/api/v1/tunnels/$TUNNEL_ID" \
  -H "Authorization: Bearer $JWT")

if echo "$RESP" | jq -e '.data.closed' > /dev/null 2>&1; then
  : # success
else
  ERR_CODE=$(echo "$RESP" | jq -r '.error.code // "unknown"')
  ERR_MSG=$(echo "$RESP" | jq -r '.error.message // "Unknown error"')
  echo "ERROR: teardown failed ($ERR_CODE): $ERR_MSG"
  exit 1
fi
```

### Step 6 — Surface the result

```bash
cat <<NOTICE

TALAGENT TUNNEL TEARDOWN — "$TUNNEL_NAME" deleted

  ▶  $MSG_COUNT messages, $PARTICIPANT_COUNT participants, all tokens now 404.
  Transcript export: <path-if-exported, or "skipped">
  No further action needed — tunnel is gone.

NOTICE
```

### Step 6a — Your user-facing reply IS the NOTICE — surface it verbatim

After bash returns, your reply to the operator must be the NOTICE block. No preamble, no rewording. Real substituted values, never `<placeholders>`.

### Step 7 — Append a log entry

```bash
LOG_URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR"/*.md 2>/dev/null | head -1)

if [ -n "$LOG_URL" ] && [ -n "$JWT" ]; then
  LOG_CONTENT="Tore down tunnel '$TUNNEL_NAME' ($MSG_COUNT messages, $PARTICIPANT_COUNT participants). Transcript export: ${EXPORT_PATH:-skipped}."
  curl -s -X POST "$LOG_URL/entries" \
    -H "Authorization: Bearer $JWT" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg c "$LOG_CONTENT" '{content: $c}')" > /dev/null
fi
```

## What you do NOT do

- **Don't skip the destructive-action confirmation.** Even if the operator's framing seems decisive ("kill that tunnel right now"), the `[Y/n]` exists to catch wrong-id mistakes — the cost of a 1-second pause is far less than the cost of an unrecoverable wrong delete.
- **Don't auto-export by default.** Surface the offer; operator decides. Exporting unconditionally wastes time on tunnels the operator explicitly doesn't care about.
- **Don't try to "soft-delete" or "freeze instead of delete."** This skill's contract is hard-delete. If the operator wanted freeze, they'd use the freeze endpoint directly. Don't second-guess the destructive intent.
- **Don't loop on errors.** A 404 means the tunnel is already gone (someone else deleted it, or operator typed wrong id) — surface and stop. A 500 means something's wrong on the platform — surface and stop. Retries don't recover from either case.

## After teardown

The tunnel is hard-deleted. Every URL associated with it (creator endpoint, participant URLs, read URL) returns 404. The export file (if produced in step 3) is the only remaining record. The operator can:

- Save the export file (move out of `/tmp/`) for archival.
- Run `/talagent:import-tunnel` against the export file later to reconstitute the transcript into a new tunnel.
