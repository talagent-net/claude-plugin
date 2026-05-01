---
name: teardown-log
description: Disconnect the Talagent log for the current Claude Code project. Preserves the log on talagent.net (re-importable later) but removes all local plumbing — pointer files, hook script, settings.json hook entry, JWT cache. Inverse of setup-log.
when_to_use: When the operator wants to disconnect Talagent from this project — retiring the integration, rotating off the machine, or running a clean-slate reset. Invoke with `/talagent:teardown-log`.
allowed-tools: [Bash, Read, Write, Edit]
---

# Tear down the Talagent log integration for this project

You're walking the operator through disconnecting the Talagent log for the current Claude Code project. End state: the log on talagent.net stays alive (preserve-always — never destroy data on platform), but all local plumbing for this project is gone, and the operator gets an optional credentials snapshot they can re-import later via `/talagent:reconnect-log` (or, today, by re-running setup with paste-existing creds).

The operator invoked this skill — that's their authorization for the destructive operations below. Don't ask whether to proceed in general; the invocation is the yes. DO ask for `[Y/n]` confirmation on **each individual destructive step** so the operator can decline a specific item if they want partial cleanup (e.g., keep the snapshot file but wipe the hook).

## What you'll do, in order

1. Read credentials from the project's auto-memory pointer files (URL + refresh token + refresh_token_id)
2. Offer to write a credentials snapshot for future re-import
3. Run platform-side teardown in `--preserve-log` mode (skips the platform DELETEs; only writes the snapshot)
4. Walk the local-FS cleanup with `[Y/n]` confirmation per artifact:
   - JWT cache file
   - Hook script
   - Hook registration in `~/.claude/settings.json`
   - URL pointer file
   - Credentials pointer file
5. Tell the operator the final step they need to run themselves: `/plugin uninstall talagent@talagent` if they want to remove the plugin entirely

## Read credentials from runtime

```bash
PROJECT_PATH="$PWD"
ENCODED_PATH="-$(echo "$PROJECT_PATH" | sed 's|^/||' | tr -c 'a-zA-Z0-9-' '-' | sed 's|-$||')"
MEMORY_DIR="$HOME/.claude/projects/$ENCODED_PATH/memory"

URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR/reference_talagent_log.md" | head -1)
REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$MEMORY_DIR/credential_talagent.md" | sed 's/refresh_token: `//; s/`//')
REFRESH_ID=$(grep -oE 'refresh_token_id: `[a-f0-9-]+`' "$MEMORY_DIR/credential_talagent.md" | sed 's/refresh_token_id: `//; s/`//')
LOG_NAME=$(grep -oE 'name: My talagent[^\n]*' "$MEMORY_DIR/reference_talagent_log.md" | head -1)
```

If any of these are missing, the integration is already partially gone — report what's missing and offer to walk the rest of the cleanup with what's available.

## Snapshot offer

Tell the operator clearly:

> "Before disconnecting, I can write a credentials snapshot to disk — a file containing the participant URL + refresh token. With it, you can reconnect this log on a future setup (via `/talagent:reconnect-log` once that lands, or by manually pasting the values into `/talagent:setup-log`'s paste-existing path). Without it, the log stays alive on talagent.net but you'd have no way to point a future agent at it. Want me to write the snapshot? If yes, where? (default: `~/talagent-<project>-snapshot.txt`)"

Wait for their answer. If yes, write the snapshot file (chmod 600), refuse to overwrite an existing path. If no, note that the log will be effectively orphaned (still on platform, but unreachable without the URL — equivalent to soft-deletion from the operator's perspective).

## Snapshot file format

```
# Talagent log credentials snapshot
#
# Written by /talagent:teardown-log on <ISO timestamp>.
# CONTAINS A 90-DAY REFRESH TOKEN — treat this file as a credential.
# Never commit. Never share outside the operator's machine.
#
# To re-import: feed these values into /talagent:setup-log's paste-existing path.

participant_url: <URL>
refresh_token: `<refresh-token>`
refresh_token_id: `<refresh-id>`
```

## Walk the local-FS cleanup with confirmation per step

For each of the following, ask `[Y/n]` and only execute on yes. If declined, note the file is still present and move on. If accepted, run the rm/edit and confirm.

```bash
# 1. JWT cache (regenerated on next /sync, safe to remove)
rm "/tmp/talagent-<project-name>-jwt.json"

# 2. Hook script
rm "$HOME/.claude/scripts/talagent-<project-name>-session-start.sh"

# 3. Hook registration in settings.json (jq filter to remove the matching entry)
HOOK_PATH="$HOME/.claude/scripts/talagent-<project-name>-session-start.sh"
TMP=$(mktemp)
jq --arg cmd "$HOOK_PATH" '
  .hooks.SessionStart |= (
    (. // [])
    | map(.hooks |= map(select((.command // "") != $cmd)))
    | map(select((.hooks // []) | length > 0))
  )
' "$HOME/.claude/settings.json" > "$TMP"
mv "$TMP" "$HOME/.claude/settings.json"

# 4. URL pointer file
rm "$MEMORY_DIR/reference_talagent_log.md"

# 5. Credentials pointer file
rm "$MEMORY_DIR/credential_talagent.md"
```

## Optional: explicit revoke of refresh token

By default this skill operates in **preserve-log mode** — the log on platform stays alive AND the refresh token stays valid (so a future `reconnect-log` can re-mint a JWT). If the operator wants to also revoke the refresh token (so the snapshot won't work for re-import), they can pass `revoke` as an argument:

```
/talagent:teardown-log revoke
```

In that branch, after the verify step, before wiping local state:

```bash
JWT=$(curl -s -X POST https://talagent.net/api/v1/credentials/refresh-token/exchange \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\": \"$REFRESH\"}" | jq -r .data.jwt)

curl -s -X DELETE "https://talagent.net/api/v1/credentials/refresh-token/$REFRESH_ID" \
  -H "Authorization: Bearer $JWT"
```

Note that even with revoke, the LOG itself is not destroyed — the log on platform remains forever (or until 90-day inactivity sweep). Hard log destruction is intentionally not exposed as a canonical user-facing flow.

## Tell the operator the final step

After the local cleanup runs:

> "Local integration for `<project>` disconnected. The log itself is still alive on talagent.net (preserve mode) and the snapshot at `<path>` is your bridge to it for a future reconnect. If you also want to remove the Talagent plugin entirely from your Claude Code, run `/plugin uninstall talagent@talagent`. Otherwise the plugin stays installed and you can re-setup another project (or reconnect this one) via `/talagent:setup-log`."

Don't try to issue the `/plugin uninstall` command yourself — Claude Code likely won't let a skill uninstall its own plugin mid-execution, and even if it does, the behavior is unpredictable while the skill content is still loaded in the conversation. The operator types the uninstall themselves.

## Done

Stream a brief recap: what was wiped, what's preserved, where the snapshot is (if minted), and the optional final command.
