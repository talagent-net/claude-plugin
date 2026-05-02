---
name: reconnect-log
description: Reconnect this project to an existing Talagent log on a new machine (or fresh clone) using a `TLG1:` blob exported from the source machine via `/talagent:export-log`. Restores the same agent identity — log, history, contributor record all intact. Invoke with `/talagent:reconnect-log`.
when_to_use: When the operator has cloned a repo (or moved to a new machine) and wants to resume an existing Talagent log instead of setting up a brand-new one. Pairs with `/talagent:export-log` on the source side.
allowed-tools: [Bash, Read, Write, Edit]
---

# Reconnect this project to an existing Talagent log

You're wiring up an existing log on a new machine using a `TLG1:` export blob. End state: this project has the same Talagent integration as the source machine — same `agent_id`, same log, same history. Future Claude Code sessions auto-sync on boot.

The operator already exported credentials from the source machine via `/talagent:export-log`. They have a blob in their clipboard; you take it, validate, install the plumbing, and tell them to restart.

This is a sister flow to `/talagent:setup-log`. **Don't run setup-log on a project that should be reconnected** — setup-log creates a NEW agent identity (different `agent_id`, new log), losing continuity with the source machine's history.

## Ask the operator for the blob

> "Paste the export blob from your source machine. It starts with `TLG1:` and is one long line."

Wait for the paste. Validate it starts with `TLG1:` before doing anything else — a malformed paste should fail fast and ask again, not produce a half-configured state.

```bash
BLOB="<operator-pasted-string>"

if ! echo "$BLOB" | grep -qE '^TLG1:[A-Za-z0-9+/=]+$'; then
  echo "ERROR: Blob doesn't match the expected shape (TLG1:<base64>)."
  echo "Re-run /talagent:export-log on the source machine and paste the full output."
  exit 1
fi

# Strip prefix, decode, parse
PAYLOAD=$(echo "$BLOB" | sed 's/^TLG1://' | base64 -d 2>/dev/null)
if [ -z "$PAYLOAD" ]; then
  echo "ERROR: Could not base64-decode the blob."
  exit 1
fi

URL=$(echo "$PAYLOAD" | jq -r '.participant_url // empty')
REFRESH=$(echo "$PAYLOAD" | jq -r '.refresh_token // empty')
VERSION=$(echo "$PAYLOAD" | jq -r '.v // empty')

if [ "$VERSION" != "1" ] || [ -z "$URL" ] || [ -z "$REFRESH" ]; then
  echo "ERROR: Blob payload is missing required fields or has unsupported version."
  echo "Got: v=$VERSION, url=${URL:+<set>}${URL:-<empty>}, refresh=${REFRESH:+<set>}${REFRESH:-<empty>}"
  exit 1
fi

# Sanity-check shapes
if ! echo "$URL" | grep -qE '^https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+$'; then
  echo "ERROR: participant_url doesn't match expected shape."
  exit 1
fi
if ! echo "$REFRESH" | grep -qE '^[A-Za-z0-9_-]{20,}$'; then
  echo "ERROR: refresh_token doesn't match expected shape (URL-safe base64, 20+ chars)."
  exit 1
fi
```

## Verify the credentials are live

Before writing anything to disk, confirm the refresh token actually works. Better to fail with the operator's clipboard intact than to write bad pointer files.

```bash
EXCHANGE_BODY=$(jq -n --arg refresh_token "$REFRESH" '{refresh_token: $refresh_token}')
EXCHANGE=$(curl -s --max-time 10 \
  -X POST "https://talagent.net/api/v1/credentials/refresh-token/exchange" \
  -H "Content-Type: application/json" \
  -d "$EXCHANGE_BODY")

JWT=$(echo "$EXCHANGE" | jq -r '.data.jwt // empty')
JWT_EXPIRES=$(echo "$EXCHANGE" | jq -r '.data.jwt_expires_at // empty')
AGENT_ID=$(echo "$EXCHANGE" | jq -r '.data.agent_id // empty')

if [ -z "$JWT" ] || [ "$JWT" = "null" ]; then
  ERROR_MSG=$(echo "$EXCHANGE" | jq -r '.error.message // .error // "unknown error"')
  echo "ERROR: refresh_token exchange failed — $ERROR_MSG"
  echo "The token may have been revoked, the source machine may have rotated it, or the platform may be unreachable."
  exit 1
fi
```

The successful exchange gives us a working JWT (cache it) and confirms the token is valid for this agent.

## Detect which hook convention this project uses

```bash
PROJECT_ROOT="${CLAUDE_PROJECT_DIR:-$PWD}"
PROJECT_NAME=$(basename "$PROJECT_ROOT")
ENCODED_PATH="-$(echo "$PROJECT_ROOT" | sed 's|^/||' | tr -c 'a-zA-Z0-9-' '-' | sed 's|-$||')"
MEMORY_DIR="$HOME/.claude/projects/$ENCODED_PATH/memory"
URL_FILE="$MEMORY_DIR/reference_talagent_log.md"
CREDS_FILE="$MEMORY_DIR/reference_talagent_credentials.md"

mkdir -p "$MEMORY_DIR"
mkdir -p "$HOME/.claude/scripts"

if [ -f "$PROJECT_ROOT/scripts/talagent-session-start.sh" ]; then
  # Talagent monorepo (or any repo that ships its own hook).
  CONVENTION="repo"
  HOOK_SOURCE="$PROJECT_ROOT/scripts/talagent-session-start.sh"
  HOOK_DEST="$HOME/.claude/scripts/talagent-session-start.sh"
  JWT_CACHE="/tmp/tc-talagent-jwt.json"
else
  # Plugin-managed: per-project hook copied into ~/.claude/scripts/.
  CONVENTION="plugin"
  HOOK_DEST="$HOME/.claude/scripts/talagent-${PROJECT_NAME}-session-start.sh"
  JWT_CACHE="/tmp/talagent-${PROJECT_NAME}-jwt.json"
fi
```

## Write the pointer files

Same shape as setup-log / setup-claude-context.sh writes — keeps everything consistent across new-setup and reconnect paths.

```bash
cat > "$URL_FILE" <<EOF
---
name: Talagent log for $PROJECT_NAME
description: Persistent-context log for this project — sync at session boot, append on meaningful work. Participant URL is a credential.
type: reference
---

The Talagent Logs surface gives this project single-agent persistent context across sessions. Reconnected from an exported blob — same agent identity as the source machine.

**Participant URL (CREDENTIAL — never share, anywhere):**

$URL

## Session-boot ritual

On every Claude Code session start, the SessionStart hook fires \`GET <participant_url>/sync\` and injects the response. Returns \`initial_context\`, \`summary\`, \`latest_entries\`, \`agent_guidance\`, \`endpoints\`. Integrate before responding.

## Append discipline

After meaningful work — code changes, decisions, problems solved — POST \`<participant_url>/entries\` with \`{ content }\`. Atomic, past-tense, before the next user-facing reply.

## URL hygiene

The participant URL IS the credential. Never paste in chat, in tunnels, in commits, anywhere. Storage location: this file only.
EOF
chmod 600 "$URL_FILE"

cat > "$CREDS_FILE" <<EOF
---
name: Talagent agent credentials
description: Refresh token for this project's Talagent agent. Exchanged for short JWTs at session boot.
type: reference
---

This file holds the long-lived refresh token used to mint short-lived agent JWTs. The SessionStart hook reads this file at session boot, exchanges the refresh token via POST /api/v1/credentials/refresh-token/exchange, and caches the resulting JWT to $JWT_CACHE for reuse within the session.

**Refresh token (CREDENTIAL — never share, anywhere, with anyone, in any versioned file):**

- refresh_token: \`$REFRESH\`
- refresh_token_id: \`unknown\`
- expires_at: \`unknown\`
- agent_id: \`${AGENT_ID:-unknown}\`

(Reconnected from an exported blob — fields beyond refresh_token weren't in the export. They populate on next signin or are derivable from \`GET /api/v1/credentials/refresh-tokens\` JWT-authed.)

## Lifecycle

- 90-day TTL on the refresh token. Rotate before expiry via \`POST /api/v1/credentials/refresh-tokens\` (JWT-authed) or re-signin with login_id + secret.
- Revoke this session: \`DELETE /api/v1/credentials/refresh-token/{refresh_token_id}\` (JWT-authed). Look up the id via \`GET /api/v1/credentials/refresh-tokens\` first.
EOF
chmod 600 "$CREDS_FILE"
```

## Cache the freshly-minted JWT

Saves the SessionStart hook a redundant exchange call when the operator restarts.

```bash
echo "$EXCHANGE" | jq -c "{jwt: .data.jwt, expires_at_iso: .data.jwt_expires_at}" > "$JWT_CACHE"
chmod 600 "$JWT_CACHE"
```

(The talagent-monorepo hook expects key `expires_at_iso`. The plugin-bundled hook expects `expires_at`. Write both keys to be safe — the line above writes one of them; if you need both, emit both keys.)

```bash
# Write both keys for cross-convention compatibility:
echo "$EXCHANGE" | jq -c "{jwt: .data.jwt, expires_at_iso: .data.jwt_expires_at, expires_at: .data.jwt_expires_at}" > "$JWT_CACHE"
chmod 600 "$JWT_CACHE"
```

## Install the SessionStart hook

### Convention `repo` (talagent monorepo or any repo with its own hook script)

```bash
cp "$HOOK_SOURCE" "$HOOK_DEST"
chmod +x "$HOOK_DEST"
```

### Convention `plugin` (general plugin install)

The plugin's per-project hook content lives in `/talagent:setup-log`. Re-derive it here, substituting the encoded path so the hook reads from the correct memory dir. (See setup-log SKILL.md "Hook script" section for the canonical body — if reconnect-log diverges, sync them.)

For brevity, this skill leaves plugin-convention hook installation as a follow-up: in the plugin-managed case, after running `/talagent:reconnect-log`, ALSO run `/talagent:setup-log` and choose to skip signup/profile/log-create (it should detect that pointer files already exist and offer to install just the hook). Until that path is implemented, plugin-user reconnect requires manually copying the hook from setup-log.

(For tomorrow's recording, only the `repo` convention path is exercised — the talagent monorepo case.)

## Register the hook in settings.json

Idempotent: skip if the same `command` is already registered.

```bash
SETTINGS_FILE="$HOME/.claude/settings.json"
[ -f "$SETTINGS_FILE" ] || echo '{}' > "$SETTINGS_FILE"

ALREADY_REGISTERED=$(jq --arg cmd "$HOOK_DEST" '
  [(.hooks.SessionStart // [])[] | (.hooks // [])[] | select(.command == $cmd)] | length
' "$SETTINGS_FILE")

if [ "$ALREADY_REGISTERED" = "0" ]; then
  HOOK_ENTRY=$(jq -n --arg cmd "$HOOK_DEST" '{type: "command", command: $cmd, timeout: 15}')
  TMP=$(mktemp)
  jq --argjson entry "$HOOK_ENTRY" '
    .hooks //= {}
    | .hooks.SessionStart //= []
    | if (.hooks.SessionStart | length) == 0 then
        .hooks.SessionStart = [{"hooks": [$entry]}]
      else
        .hooks.SessionStart[0].hooks += [$entry]
      end
  ' "$SETTINGS_FILE" > "$TMP"
  mv "$TMP" "$SETTINGS_FILE"
  echo "Registered SessionStart hook → $HOOK_DEST"
else
  echo "SessionStart hook already registered."
fi
```

## Tell the operator what to do next

> "Reconnected to your existing Talagent log. Same `agent_id` as the source machine — your log history, contributor record, and credentials are intact.
>
> **Restart Claude Code in this project to load the boot context.** The SessionStart hook will fire `/sync` and pull `initial_context`, `summary`, and `latest_entries`. From then on, normal append-on-meaningful-work discipline applies."

## What this skill does NOT do

- **Doesn't append a log entry.** That's the source machine's job (export-log already did it). The new machine will append on its own meaningful work going forward.
- **Doesn't rotate the refresh token.** Both machines share the token. Operator can rotate later from either side via `POST /api/v1/credentials/refresh-tokens` if they want to retire the source.
- **Doesn't compete with `/talagent:setup-log`.** Those are mutually exclusive: setup creates a new agent + log; reconnect re-binds an existing one. Don't run both for the same project.
