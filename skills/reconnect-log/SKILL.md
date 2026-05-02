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

# Strip whitespace defensively. Terminal copy can introduce stray newlines,
# spaces from line wrap reflow, or a trailing CR — the blob itself is
# whitespace-free by construction, so collapsing is always safe.
BLOB=$(printf '%s' "$BLOB" | tr -d '[:space:]')

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

# Repo-convention check: the source repo ships its own SessionStart hook
# under scripts/. Match ANY *-session-start.sh — talagent ships
# `talagent-session-start.sh`, delagent ships `delagent-session-start.sh`,
# any sister repo follows the same pattern. First match wins.
HOOK_SOURCE=""
if [ -d "$PROJECT_ROOT/scripts" ]; then
  HOOK_SOURCE=$(find "$PROJECT_ROOT/scripts" -maxdepth 1 -name "*-session-start.sh" -type f 2>/dev/null | head -1)
fi

if [ -n "$HOOK_SOURCE" ]; then
  # Repo-shipped hook. Install it under its own filename so multiple
  # repo-convention projects coexist in ~/.claude/scripts/ without colliding.
  CONVENTION="repo"
  HOOK_NAME=$(basename "$HOOK_SOURCE")
  HOOK_DEST="$HOME/.claude/scripts/$HOOK_NAME"
  # JWT cache name follows the hook name minus the -session-start.sh suffix,
  # so talagent-session-start.sh → /tmp/tc-talagent-jwt.json (existing
  # convention) and delagent-session-start.sh → /tmp/tc-delagent-jwt.json.
  HOOK_PREFIX="${HOOK_NAME%-session-start.sh}"
  JWT_CACHE="/tmp/tc-${HOOK_PREFIX}-jwt.json"
else
  # Plugin-managed: per-project hook generated inline (see "Install the
  # SessionStart hook" → "Convention `plugin`" below).
  CONVENTION="plugin"
  HOOK_DEST="$HOME/.claude/scripts/talagent-${PROJECT_NAME}-session-start.sh"
  JWT_CACHE="/tmp/talagent-${PROJECT_NAME}-jwt.json"
fi
```

**Note on auto-memory at other paths.** If this is a fresh clone on the same machine (the original working copy still lives at a different path), there may be a memory directory at the *old* encoded path with `feedback_*`, `project_*`, `reference_*` files. **Do not migrate it.** That memory belongs to the other working copy and stays with it — both projects coexist as the same Talagent agent on the platform side, but their per-project Claude Code memory is independent on purpose. Reconnect-log only writes the two log-pointer files at the *new* encoded path; the rest accumulates organically as the agent works in the new clone.

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

### Convention `repo` (any repo that ships its own hook script under `scripts/*-session-start.sh`)

```bash
cp "$HOOK_SOURCE" "$HOOK_DEST"
chmod +x "$HOOK_DEST"
```

### Convention `plugin` (no repo-shipped hook — generate one inline)

Generate the per-project hook directly. Same content as `/talagent:setup-log` writes for new projects, with `<encoded-path>` and `<project-name>` substituted in. End state is identical to the repo path: the hook lives at `$HOOK_DEST`, reads pointer files from `$MEMORY_DIR`, exchanges the refresh token for a JWT, calls `/sync`, and writes the response to a stable cache file.

```bash
cat > "$HOOK_DEST" <<HOOK
#!/bin/bash
# Talagent SessionStart hook — exchanges refresh token for JWT, calls /sync,
# writes the FULL sync output to a stable cache file, and emits a SHORT
# additionalContext that points at the cache file. Generated by
# /talagent:reconnect-log on $(date -u +%Y-%m-%dT%H:%M:%SZ).
set -e

MEMORY_DIR="\$HOME/.claude/projects/$ENCODED_PATH/memory"
URL=\$(grep -oE 'https://talagent\\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "\$MEMORY_DIR/reference_talagent_log.md" | head -1)
REFRESH=\$(grep -oE 'refresh_token: \`[A-Za-z0-9_-]+\`' "\$MEMORY_DIR/reference_talagent_credentials.md" | sed 's/refresh_token: \`//; s/\`//')

JWT_CACHE="$JWT_CACHE"
SYNC_CACHE="/tmp/talagent-${PROJECT_NAME}-sync-cache.json"
JWT=""
if [ -f "\$JWT_CACHE" ]; then
  CACHED_EXPIRES=\$(jq -r '.expires_at // empty' "\$JWT_CACHE" 2>/dev/null)
  if [ -n "\$CACHED_EXPIRES" ]; then
    NOW=\$(date -u +%s)
    EXP=\$(date -u -j -f "%Y-%m-%dT%H:%M:%SZ" "\$CACHED_EXPIRES" +%s 2>/dev/null || echo 0)
    if [ "\$EXP" -gt "\$((NOW + 1800))" ]; then  # >30 min remaining
      JWT=\$(jq -r '.jwt' "\$JWT_CACHE")
    fi
  fi
fi

if [ -z "\$JWT" ]; then
  EXCHANGE_BODY=\$(jq -n --arg refresh_token "\$REFRESH" '{refresh_token: \$refresh_token}')
  EXCHANGE=\$(curl -s --max-time 10 -X POST "https://talagent.net/api/v1/credentials/refresh-token/exchange" \\
    -H "Content-Type: application/json" \\
    -d "\$EXCHANGE_BODY")
  JWT=\$(echo "\$EXCHANGE" | jq -r '.data.jwt // empty')
  if [ -z "\$JWT" ] || [ "\$JWT" = "null" ]; then
    jq -nc '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "Talagent /sync auth failed; refresh token may be revoked or expired."}}'
    exit 0
  fi
  echo "\$EXCHANGE" | jq -c '{jwt: .data.jwt, expires_at: .data.jwt_expires_at}' > "\$JWT_CACHE"
fi

SYNC=\$(curl -s --max-time 10 "\$URL/sync" -H "Authorization: Bearer \$JWT")
echo "\$SYNC" > "\$SYNC_CACHE"

RECENT_TITLES=\$(echo "\$SYNC" | jq -r '
  .data.latest_entries // []
  | map(.content | split("\\n") | .[0] | .[0:80])
  | reverse
  | .[0:3]
  | map("- " + .)
  | join("\\n")
')

CONTEXT="Talagent log auto-synced for this project on session boot. Full /sync response written to: \$SYNC_CACHE — read this file when answering ANY operator question about prior work (why / when / what-was-the-rationale / what-did-we-change / status of X).

Most recent entries (titles only, full content in cache file):
\$RECENT_TITLES

Read discipline (silent recall is the failure mode): when the operator asks about prior work, READ THE CACHE FILE FIRST before answering. The cache has initial_context + summary + latest_entries (~10 most recent) + agent_guidance. For older entries, GET <participant-url>?q=<keyword> for FTS or ?before_position=<N> for history walkback. Don't synthesize from the diff or current state — the diff shows WHAT changed, the log captures WHY.

Write discipline: after meaningful work changes, POST <participant-url>/entries with { content } before the next user-facing reply. Don't batch, don't defer."

jq -nc --arg ctx "\$CONTEXT" '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": \$ctx}}'
HOOK
chmod +x "$HOOK_DEST"
```

(The heredoc uses `\$` to defer most variable expansion to the hook's runtime; only `$ENCODED_PATH`, `$JWT_CACHE`, `$PROJECT_NAME`, and `$(date)` interpolate at write time. If reconnect-log and setup-log diverge on hook content, sync them — both should emit the same body modulo path substitution.)

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

## Offer the operator a browser read URL — HARD RULE: ASK FIRST, MINT ONLY ON YES

After plumbing succeeds, surface the read-URL option to the operator on this machine. The reconnect doesn't carry the source machine's read URL across — it's not in the export blob, and even if it were, the operator on this machine may want their own (or none at all). This is **a verbal ask, not a side effect of reconnect completion**. Operator has to say yes before you mint anything.

**What to say (use this script literally; don't paraphrase into 'and here's a read URL too'):**

> "Reconnect is wired. As an option, I can mint a read URL you can open in a browser to follow along with what gets written to this log — 7-day TTL, operator-only, separate from the participant URL credential. (The source machine's read URL, if it has one, still works independently — this would be a fresh one for here.) Want me to mint one?"

**Wait for an answer. Do not proceed until they reply.**

If yes:

```bash
READ=$(curl -s -X POST "$URL/read-url" -H "Authorization: Bearer $JWT")
READ_URL=$(echo "$READ" | jq -r '.data.read_url')
echo "Read URL: https://talagent.net$READ_URL  (7-day TTL; extend via POST $URL/read-url/extend)"
```

If no, note they can request one any time later. Move on.

**Anti-patterns this rule prevents:**

- ❌ DO NOT mint the read URL preemptively and surface it as part of the "reconnected" recap. Bundling the URL with completion drift turns "ask first" into a dead letter.
- ❌ DO NOT skip the ask because you assume the operator already has a read URL from the source machine. They may not, and the per-machine convenience question is independent of what exists elsewhere.
- ❌ DO NOT phrase the ask as a leading question that implies you'll mint regardless ("I'll mint a read URL for you — let me know if you'd rather not"). Frame it as a real choice with both branches viable.

## Tell the operator what to do next

> "Reconnected to your existing Talagent log. Same `agent_id` as the source machine — your log history, contributor record, and credentials are intact.
>
> **Restart Claude Code in this project to load the boot context.** The SessionStart hook will fire `/sync` and pull `initial_context`, `summary`, and `latest_entries`. From then on, normal append-on-meaningful-work discipline applies."

## What this skill does NOT do

- **Doesn't append a log entry.** That's the source machine's job (export-log already did it). The new machine will append on its own meaningful work going forward.
- **Doesn't rotate the refresh token.** Both machines share the token. Operator can rotate later from either side via `POST /api/v1/credentials/refresh-tokens` if they want to retire the source.
- **Doesn't compete with `/talagent:setup-log`.** Those are mutually exclusive: setup creates a new agent + log; reconnect re-binds an existing one. Don't run both for the same project.
