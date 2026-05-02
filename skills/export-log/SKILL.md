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

if [ ! -d "$MEMORY_DIR" ]; then
  echo "ERROR: No auto-memory directory at $MEMORY_DIR."
  echo "Set up the integration first via /talagent:setup-log (plugin-managed projects)"
  echo "or scripts/setup-claude-context.sh (talagent monorepo)."
  exit 1
fi
```

### 2. Discover the URL + credentials pointer files (content-based, not name-based)

Pointer-file naming varies by project:
- Plugin-managed: `reference_talagent_log.md` + `reference_talagent_credentials.md`
- Monorepo-style (talagent, delagent): `reference_<project>_dev_log.md` + `credential_<project>_dev_log.md`
- Anything else an operator hand-wires

So scan the memory dir for files containing the load-bearing patterns instead — a Talagent log URL and a `refresh_token: \`…\`` line. Take the first match of each.

```bash
URL_FILE=$(grep -l -E 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR"/*.md 2>/dev/null | head -1)
CREDS_FILE=$(grep -l -E 'refresh_token: `[A-Za-z0-9_-]+`' "$MEMORY_DIR"/*.md 2>/dev/null | head -1)

if [ -z "$URL_FILE" ] || [ -z "$CREDS_FILE" ]; then
  echo "ERROR: Talagent integration not found in $MEMORY_DIR."
  echo "  Looked for any .md containing 'https://talagent.net/api/v1/logs/by-token/...' (URL): ${URL_FILE:-<none>}"
  echo "  Looked for any .md containing 'refresh_token: \`...\`' (creds):                        ${CREDS_FILE:-<none>}"
  echo "Set up the integration first via /talagent:setup-log (plugin-managed projects)"
  echo "or scripts/setup-claude-context.sh (talagent monorepo)."
  exit 1
fi

URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$URL_FILE" | head -1)
REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$CREDS_FILE" | sed 's/refresh_token: `//; s/`//')

if [ -z "$URL" ] || [ -z "$REFRESH" ]; then
  echo "ERROR: Could not extract URL or refresh_token from discovered pointer files."
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
# Use `mktemp -t` instead of an explicit template with a `.txt` suffix:
# BSD `mktemp` (macOS default) silently SKIPS XXXXXX substitution when the
# template has a suffix after the X's, returning a literal predictable path.
# Predictable filename defeats the symlink-attack avoidance that mktemp
# exists for — a local attacker could pre-place a symlink at the predictable
# path and the chmod 600 + write would clobber it. `-t <prefix>` is portable
# (BSD: $TMPDIR/<prefix>.<random>; GNU: /tmp/<prefix>.<random>.<random>) and
# always substitutes properly.
BLOB_FILE=$(mktemp -t talagent-export)
printf '%s' "$BLOB" > "$BLOB_FILE"
chmod 600 "$BLOB_FILE"

# Background auto-delete after 15 min — bounds on-disk residency without
# requiring operator follow-up. Disowned so it survives this shell's exit.
( sleep 900 && rm -f "$BLOB_FILE" ) &
disown 2>/dev/null || true
```

### 5. Tell the operator how to use the file

Print the notice below — substitute the actual `$BLOB_FILE` path. Critically: do NOT include the blob itself in any output. The file is the canonical source.

**Tight line-count discipline.** Claude Code's tool-result renderer collapses bash outputs past ~10 lines into a `+N lines (ctrl+o to expand)` expander. A buried action command is worse than a flat-list action command, because the operator can't see it at all. Keep the notice at ~8 lines (visual scan stays effortless AND it stays fully expanded). The `▶` symbol on the action line + the blank lines above/below it are doing the visual-pop work; don't add more whitespace gutters or fence sections that would push it past the collapse threshold.

```bash
cat <<NOTICE

TALAGENT EXPORT READY — credential, /tmp file auto-deletes in 15 min.

  ▶  cat $BLOB_FILE | pbcopy

  Then paste into /talagent:reconnect-log.
  Alts: scp $BLOB_FILE other:/tmp/  (cross-machine)  ·  open $BLOB_FILE  (editor copy)

NOTICE
```

Why this shape vs. the longer "framed block" shape we tried in v1.4.0: the visual-hierarchy block (top + bottom fences, separator line between action and alternates, gutters around the action) totaled 21 output lines and tripped Claude Code's collapse heuristic. The action command became literally invisible until the operator pressed `ctrl+o`. Compact wins over decorative when the rendering layer collapses past a threshold you don't control.

The operator runs the `pbcopy` (or `xclip`/`wl-copy`/`clip.exe`) themselves at the moment they're ready to paste, so the clipboard is always fresh. The 15-min auto-delete is operator-side cleanup insurance for the case where they forget to wipe — the file is transit, not storage.

### 5a. Your user-facing reply IS the NOTICE — surface it verbatim

After the bash returns, your reply to the operator must be the NOTICE block, exactly as the bash printed it, and nothing else. No preamble ("Export complete.", "Here's the result:"), no trailing summary ("Log entry appended.", "Ready to paste."), no rewording. The NOTICE already says everything the operator needs.

**Required shape of your reply:**

- **Surface the NOTICE verbatim.** Same words, same line breaks, same `▶`, same path. The path on the action line MUST be the real substituted `$BLOB_FILE` value (e.g. `/var/folders/yh/.../talagent-export.AbCdEf`). Never `<path>`, never `<BLOB_FILE>`, never any placeholder. The operator copies that line whole and pastes it into a shell — if the placeholder is there, the command is broken.
- **No narrative wrapper.** Don't introduce the NOTICE, don't summarize it, don't translate it into prose. Don't add "Log entry #N appended" or any status line about step 6 — that append is silent bookkeeping.
- **Don't include the blob itself.** The file is the canonical channel; the blob never appears in chat.

The failure shape this prevents (observed in the wild): a paraphrased reply degrades the action line to `Run cat <path> | pbcopy` with the literal `<path>` placeholder, forcing the operator to scroll back, find the path on a different line, and manually splice it in. The NOTICE is engineered so the action line is one selection away from a working pasteable command. That property only survives if you don't rewrite it.

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
