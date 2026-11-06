# Dropbox MCP Skill Doctrines (SKILL.md)

Doctrines for working with Dropbox from PAVE through the **remote Dropbox MCP
server** (`https://mcp.dropbox.com/mcp`, bridged as `mcp_dropbox_*` tools).

Every claim carries a tag. Trust them differently:

| Tag | Meaning |
|---|---|
| `[verified <date>]` | Tested by PAVE against the live server/account. |
| `[PAVE source]` | Read directly from openpave server source (line-cited). Behavior of PAVE, not of Dropbox. |
| `[web]` | From Dropbox help pages or third-party writeups (usecarly.com, scalekit.com, 2026-07). Not independently reproduced. |
| `[untested]` | No evidence either way. Do not rely on it. |

If you change something in this file, replace blanket dates with per-claim
tags. A single "verified" stamp over unverified material launders guesswork
into fact.

---

## 0. What this skill is — and is not

- Doctrine-only skill: entrypoint is this file, no local code, **no sandbox
  applies**. The Dropbox MCP is a remote HTTPS service; no Dropbox code runs
  on your machine. `[verified 2026-11-06 — package layout + PAVE MCP architecture]`
- The MCP token never enters a skill sandbox or process env: openpave R6
  attaches it only inside the PAVE server's `connectServer()`.
  `[PAVE source — server/mcp-manager.js R6, lines 10-13]`
- **The CLI skill's credential does not transfer.** The sandboxed `dropbox`
  skill's `DROPBOX_REFRESH_TOKEN` belongs to the CNR broker's OAuth app; the
  MCP server grants only its own app. Never paste one into the other's slot.
  `[verified by construction — two different OAuth client registrations]`

## 1. Installing the Dropbox MCP in PAVE

The remote server is Streamable-HTTP at `https://mcp.dropbox.com/mcp`.
`[web]` Open beta since March 2026. `[web]`

```
capabilities_add https://mcp.dropbox.com/mcp
```

- PAVE derives the entry id from the host: `mcp.dropbox.com` → `dropbox`,
  so bridged tools are named `mcp_dropbox_*` and the token key is
  `MCP__dropbox__ACCESS`. `[PAVE source — capabilities-custom-routes.js host→id rule]`
- Expect first connect to end `status: error` until auth completes
  (`401` unauthorized, or `429` — the edge rate-limits bare probes;
  observed on `GET /` and `/.well-known/...` 2026-11-06). A dead-looking
  entry at this stage is normal. `[verified 2026-11-06 — bare probe 429; error → installed after OAuth]`
- **The repair verb is `capabilities_refresh dropbox`.** Never re-run
  `capabilities_add` to fix a broken connection — it rewrites the registry
  entry with `auth.type: none` and can destroy the auth config
  (`openpave-wrike-mcp` doctrine, same bridge). `[PAVE source + wrike precedent]`
- After connect, set `auth.type: "oauth2"` on the `servers.json` entry
  (read-modify-write; `capabilities_add` leaves it `none`), then run the
  OAuth start below. `[verified 2026-11-06]`

### R10 first-use grant — the enum trap

Bridged calls are refused with `permissionAsk: true` until
`mcp:dropbox` is allowed. The server-side `permission` tool supports
`type: "mcp"` (`tools/permission.js:1270`), but **the agent-surface tool
schema enum omits `mcp` and the client coerces the value to `module`**,
which writes a useless `modules` grant. `[verified 2026-11-06 — three coerced
calls; modules:* already present made them all no-ops]`

Working paths, in order: `[verified 2026-11-06]`

1. `permission` tool `action: "grant"`, `type: "mcp"`, `target: "mcp:dropbox"`
   — works from surfaces whose schema passes `mcp` through; shows the user
   a confirmation card.
2. Direct edit of `~/.pave/permissions.yaml` (read-modify-write):

   ```yaml
   permissions:
     mcp:
       allowed:
         - mcp:dropbox
   ```

   `mcp-config-store.js` re-reads on mtime change — no restart needed.
   `[PAVE source — mcp-config-store.js:260 mcpPermissionDecision, mtime cache]`

## 2. Auth

Dropbox's MCP surface is **OAuth-only** — no API key, no M2M/client-credentials
flow; every token traces to a user's browser consent. `[web]`

**Outcome 1 is live: DCR accepted, full OAuth lane works.**
`[verified 2026-11-06]` Discovery chain `www-authenticate+prm+as`
(RFC 9728 PRM via the `WWW-Authenticate` header), RFC 7591 DCR issued PAVE
its own client id (`ydww2fwnzkxganl`), PKCE S256, RFC 8707 resource parameter,
scopes `account_info.read files.* file_requests.* sharing.*`. Tokens land as
`MCP__dropbox__ACCESS/REFRESH` in `~/.pave/tokens.yaml`; the entry ends
`status: installed` with `audienceOrigin: https://mcp.dropbox.com` pinned.
The "trusted DCR client list" restriction from third-party writeups did not
apply to the live server. `[web claim contradicted by direct test]`

The other two outcomes, kept for fallback if Dropbox changes behavior:

2. **DCR refused** → pin your own app: register in the Dropbox App Console,
   set `auth.oauth.clientId` (+ `authorizeUrl`/`tokenUrl` pins) on the
   `servers.json` entry. Resolution order: persisted DCR id > registry pin >
   fresh DCR. `[PAVE source — mcp-oauth-flow.js:403-406]`
3. **Confidential client demanded** → PAVE refuses the flow before minting
   state (#3110 guard) and names the bearer path. Bearer is a poor fit:
   Dropbox access tokens are short-lived (~4 h). `[PAVE source — mcp-oauth-flow.js:397-400]` `[web — token lifetime]`

Agent-driven OAuth start (no browser session on the agent side needed — the
user opens the `authorizeUrl` anywhere):

```bash
curl -s -X POST http://127.0.0.1:4096/capabilities/mcp/dropbox/oauth/start \
  -H "x-pave-internal-token: $(cat ~/.pave/.internal-token)" \
  -H "x-pave-internal-user: <owner email>" \
  -H 'Content-Type: application/json' -d '{}'
```

Sbx note: unlike the CLI skill's brokered OAuth (local-only, 403 behind the
sbx gateway), **MCP OAuth start and bridged execution both pass the pinned
sbx member**. `[PAVE source — #3106 arms: mcp-oauth-flow.js:338, mcp-manager.js:784]`

**Hosted-instance caveat for Outcomes 1-3.** The default `redirect_uri` is
`http://127.0.0.1:<PAVE_PORT|4096>/oauth/callback` — a browser running outside
the container cannot reach it, so the consent redirect dies and no code ever
reaches `/oauth/callback`. Set `PAVE_PUBLIC_BASE_URL` in `settings.yaml` to an
origin that resolves to PAVE's own server (the callback is a route on PAVE,
not on a proxy), or use Outcome 4, which needs no callback at all. The base is
deliberately never derived from `X-Forwarded-Proto`/`Host` — a spoofable header
must not be able to steer the auth code elsewhere. `[PAVE source — resolveRedirectUri, mcp-oauth-flow.js:107-121]`

### Outcome 4 — no-redirect manual lane (containers, hosted instances)

Use when discovery cannot complete (see §8) or the callback cannot reach PAVE.
**Verified end to end 2026-11-06** in a Debian 12 sbx container where the
native lane could not work: app created, authorize URL hand-built, user
consented, code pasted back, exchange `http=200`, tokens stored, bridge
verified with `who_am_i` + `get_usage_and_quota`. `[verified 2026-11-06]`

Dropbox makes `redirect_uri` optional on the code flow: when it is omitted, the
authorization code is displayed on dropbox.com for the user to copy.
`[web — Dropbox OAuth guide, "Implementing Code Flow"]` That removes both
blockers at once — no PRM discovery, no callback listener.

**This lane cannot use DCR.** Dynamic client registration rejects a request
with no `redirect_uris`: `{"error":"invalid_request","error_description":
"redirect_uris is required"}` `http=400`. So the manual lane always needs your
own Developer Console app; there is no way to borrow a registered client.
`[verified 2026-11-06]`

1. Create your own app: Developer Console → **Full Dropbox** → scopes
   `account_info.read files.metadata.read files.content.read
   files.content.write sharing.read file_requests.read` → Settings → enable
   **Allow public clients** (PKCE) → leave **Redirect URIs empty**.
   Do NOT reuse a DCR-issued id for this lane: a DCR client is registered
   *against* a specific `redirect_uri`, so omitting the parameter may be
   refused, and the id does not move between machines.
   `[PAVE source — registerDynamicClient passes redirectUri, mcp-oauth-flow.js:404-406]`
2. Build the authorize URL by hand. Note the absent `redirect_uri`, and
   `token_access_type=offline` — without it Dropbox returns NO refresh token
   and you re-consent every ~4 h. `[web — Dropbox OAuth guide]`

   ```
   https://www.dropbox.com/oauth2/authorize
     ?client_id=<APP_KEY>
     &response_type=code
     &token_access_type=offline
     &code_challenge_method=S256
     &code_challenge=<base64url(sha256(verifier))>
     &scope=account_info.read files.metadata.read files.content.read
            files.content.write sharing.read file_requests.read
     &state=<random>
   ```
   (Wrapped for readability — join into a single line before use. The `scope`
   value is space-separated and must be URL-encoded.)
3. Inspect the URL before using it: decode `scope`, confirm
   `token_access_type=offline` and `code_challenge_method=S256` are present.
   A malformed URL costs a consent round-trip to discover otherwise. `[web]`
4. Sign in on any device. Dropbox shows the code on the page. Copy it.
5. Exchange the code:

   ```
   POST https://api.dropboxapi.com/oauth2/token
   grant_type=authorization_code
   code=<CODE>
   client_id=<APP_KEY>
   code_verifier=<VERIFIER>
   ```

   The guide phrases PKCE as passing `code_verifier` *instead of* `client_id`.
   Send **both** — verified accepted, `http=200`, all scopes granted.
   `[verified 2026-11-06 — access_token 1416 chars, refresh_token 64 chars,
   expires_in 14400 s]`
6. Store the tokens with the built-in route — it writes
   `MCP__dropbox__ACCESS` / `MCP__dropbox__REFRESH` into `~/.pave/tokens.yaml`
   (no helper script needed, unlike the upstream Google Workspace bridge):

   ```bash
   curl -X PUT http://127.0.0.1:4096/capabilities/mcp/dropbox/tokens \
     -H "x-pave-internal-token: $(cat ~/.pave/.internal-token)" \
     -H "x-pave-internal-user: <owner email>" \
     -H 'Content-Type: application/json' \
     -d '{"access":"<access_token>","refresh":"<refresh_token>"}'
   ```

   Returns `{"id":"dropbox","stored":true,"audienceOrigin":"https://mcp.dropbox.com"}`
   — the `audienceOrigin` pin is set for you here, no separate step.
   `[PAVE source — capabilities-routes.js:362]` `[verified 2026-11-06]`
7. Pin the refresh lane. The native flow writes `oauth.tokenEndpoint` as a side
   effect; skipping the flow leaves it absent, and `refreshEligible` then
   returns false — the entry silently never refreshes. It requires all three:
   `auth.type === 'oauth2'`, `oauth.tokenEndpoint`, and a stored
   `MCP__dropbox__REFRESH`. Keep `oauth2`; `bearer` is static and cannot
   refresh. `[PAVE source — mcp-oauth-flow.js:417-421; mcp-manager.js:440-446]`

   ```json
   "auth": { "type": "oauth2", "oauth": {
     "clientId": "<APP_KEY>",
     "tokenEndpoint": "https://api.dropboxapi.com/oauth2/token",
     "authorizeUrl": "https://www.dropbox.com/oauth2/authorize",
     "tokenUrl": "https://api.dropboxapi.com/oauth2/token"
   }}
   ```
8. Run `capabilities_refresh dropbox`. Mandatory, not optional: the manager
   attaches the token at connect time, so a live connection keeps the old one.
   Returns `Refreshed dropbox — status installed`.
   `[PAVE source — mcp-manager.js R6]` `[verified 2026-11-06]`
9. Grant R10 (§1 above), then verify with `who_am_i` and
   `get_usage_and_quota`.

Note the shape difference from the Google Workspace playbook this lane is
adapted from: Google's manual mode renders a finished credential blob in the
page, so it needs only a store step. Dropbox renders a bare authorization
**code**, so step 5 (the exchange) is unavoidable.

## 3. Tools — 45, snapshot-verified

`[verified 2026-11-06 — live tools/list snapshot in servers.json, all 45 rows
carry inputSchema; e2e calls who_am_i / get_usage_and_quota / list_folder
succeeded through the bridge]` Third-party writeups said ~23 file tools;
the live surface is larger — Paper and Sign are included.

| Group | Tools |
|---|---|
| Core files (15) | `who_am_i`, `get_usage_and_quota`, `list_folder`, `search`, `get_file_metadata`, `get_file_content`, `create_file`, `create_folder`, `copy`, `move`, `delete`, `download_link`, `list_file_revisions`, `restore_file_revision`, `get_file_content_metadata` |
| Sharing (3) | `create_shared_link` (private view-only; **cannot** make public/anyone-links — `INVALID_AUDIENCE`), `get_shared_link_metadata`, `list_shared_links` |
| File requests (3) | `create_file_request`, `get_file_request`, `list_file_requests` |
| History/rewind (2) | `list_restore_events`, `restore_folder` |
| Extraction/transform (2) | `run_file_content_extraction` (markdown/text/OCR/transcript/keyframes), `transform_file` (PDF/HTML convert, thumbnails, video frames — nothing written to Dropbox) |
| Catalog bridge (4) | `list_dropbox_operations`, `get_dropbox_operation`, `run_read_dropbox_operation`, `run_write_dropbox_operation` — full Dropbox API surface via operation ids; MUST `get_dropbox_operation` before `run_*` |
| Async (1) | `check_job_status` — polls move/copy/delete/restore_folder jobs |
| Paper (10) | `paper_list_recent_docs`, `paper_read_document`, `paper_create_document`, `paper_insert`, `paper_insert_image`, `paper_edit_content`, `paper_edit_table`, `paper_apply_markdown`, `paper_comment`, `paper_get_doc_history` |
| Sign (5) | `sign_list_signature_requests`, `sign_create_signature_request_draft`, `sign_download_signature_request`, `sign_send_signature_request_reminders`, `sign_cancel_signature_request` |

Dump the live surface yourself:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema <tool>
```

## 4. Path & ID doctrine

`[verified 2026-11-06 — live schemas + live list_folder/who_am_i responses]`

- **Three identifier forms** accepted by path params: fq_path (`/Reports/q3.pdf`),
  ns_path (`ns:14397394131//Reports`), file_id (`id:yFBSsv4HM_...`), plus
  `id:<id>/child/path` composites. Dropbox shared/web URLs are NOT accepted.
- **Responses return ns_path form** (`ns:N//name`), not fq_path — keep what a
  call returns and feed it back verbatim; don't hand-convert between forms.
- `who_am_i` gives the namespace map: `root_namespace_id` ≠
  `home_namespace_id` on team accounts (root = team space, home = member
  folder). Mounted team folders appear in listings as `object_type: "mount"`.
- `object_type` values: `mount` | `folder` | `file`. Paper docs list with a
  `.paper` extension and a placeholder `size: 200` — that is not real bytes.
- **`list_folder` is RECURSIVE BY DEFAULT** (`recursive: true or omitted
  lists descendants recursively`). An unbounded `{"path": ""}` on a real
  account returns the whole tree — pass `recursive: false` and/or
  `max_results` (1–600) unless you mean it. `[verified 2026-11-06 — root call
  returned nested tree across 4 namespaces]`
- Pagination is cursor-based (`cursor` in, `cursor` out) on `list_folder`,
  `search`, `list_shared_links`, `list_file_requests`, `list_restore_events`.
- **Do not reuse a cursor from a response you did not fully read.** If the
  bridge truncated the payload (a long recursive listing, for example), re-issue
  the call rather than continuing from that cursor — the cursor belongs to a
  page whose contents you never saw, and the gap is silent.
  `[verified 2026-11-06 — recursive list_folder on a real tree returned entries
  plus a cursor while the transcript was truncated; re-issuing was the fix]`

## 5. Diagnostic fallback: drive the server directly

The bridge can swallow real error text (wrike bridge defect §0A; re-check
here). When a bridged call fails with a bare `isError`, use the committed
probe — it is in THIS repo, so unlike the wrike probe it cannot be wiped with
`~/.pave/cache`:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema get_file_metadata
node probe/dropbox-probe.mjs call search '{"query":"invoice","max_results":10}'
```

The probe reads the token from `~/.pave/tokens.yaml` — never from argv, never
echoed. Behind a proxy, prefix with `NODE_USE_ENV_PROXY=1`.

Bridge behavior observed so far: results arrive as a raw JSON string payload
(the MCP layer stringifies structured content). Parse it; don't expect pretty
objects. `[verified 2026-11-06 — who_am_i/quota/list_folder all stringified]`

## 6. Config reference

| Key | Where | Purpose | Status |
|---|---|---|---|
| `MCP__dropbox__ACCESS` | `~/.pave/tokens.yaml` | MCP access token (0600) | `[verified 2026-11-06]` |
| `MCP__dropbox__REFRESH` | `~/.pave/tokens.yaml` | MCP refresh token | `[verified 2026-11-06]` |
| `auth.type` | `~/.pave/mcp/servers.json` | `oauth2` (OAuth path) or `bearer` (fallback) | `[verified 2026-11-06 — oauth2 live]` |
| `oauth.registeredClientId` | `~/.pave/mcp/servers.json` | DCR-issued client id (`ydww2fwnzkxganl`) | `[verified 2026-11-06]` |
| `audienceOrigin` | `~/.pave/mcp/servers.json` | R6 pin — token never replays to another origin | `[verified 2026-11-06 — set after callback]` |
| `mcp:dropbox` | `~/.pave/permissions.yaml` → `permissions.mcp.allowed` | R10 first-use approval for the bridge | `[verified 2026-11-06 — grant unblocked who_am_i]` |

## 7. Open questions

1. Does the doubled-result bridge defect (wrike §0B) reproduce here?
   `[untested]`
2. Does the swallowed-error-text bridge defect (wrike §0A) reproduce here?
   `[untested]`
3. Does the token refresh lane work unattended past the first ~4 h access
   token expiry (no forced re-consent)? `[untested — first e2e ran inside
   one token lifetime]`
4. ~~Does Outcome 4 work end to end, and does the exchange want `client_id`,
   `code_verifier`, or both?~~ **Closed 2026-11-06 — YES.** Lane verified end
   to end; the exchange accepts **both** `client_id` and `code_verifier`.
5. Does the refresh ladder survive with only a pinned `oauth.tokenEndpoint`
   (Outcome 4 step 7), i.e. without a `discovery` field on the entry?
   Partly answered: the pins make the entry refresh-**eligible** and
   `capabilities_refresh` returns `installed`, but the 4 h expiry crossing
   itself was still pending when this was written. `[untested — check pending]`

Closed 2026-11-06: PRM discovery + public-client AS admission (was Q1) —
YES, via `www-authenticate`; DCR acceptance (was Q2) — YES, own client id
issued; live tool surface (was Q3) — 45 tools, table in §3; Outcome 4 (was Q4)
— YES, verified end to end, and DCR cannot be used for it (`redirect_uris is
required`).

## 8. Troubleshooting — "no protected-resource metadata" in a container

Symptom: `oauth/start` returns

```
OAuth discovery failed for https://mcp.dropbox.com:
no protected-resource metadata at https://mcp.dropbox.com
```

while `curl` reaches the same URLs fine from the same container. `[verified 2026-11-06 — reproduced on a Debian 12 sbx container]`

**Cause: the SSRF guard, not Dropbox.** Every discovery fetch is gated by the
fetch guard, which resolves the hostname with `dns.promises.lookup` and rejects
private ranges. Inside an sbx container DNS is a catch-all that answers every
hostname with the egress gateway (`10.212.<N>.2` — inside `10.0.0.0/8`), so the
guard throws *before the request is sent*. The gateway is exempted only when
`SBX_GW_IP` is present in the **server's** env, and only the sbx driver sets it
(the model cannot set process env). `[PAVE source — fetch-ssrf-guard.js:299, fetch-egress-gateway.js:26-31]`

**Confirmed on the affected instance 2026-11-06.** `[verified 2026-11-06]`

```
tr '\0' '\n' < /proc/1/environ | grep -E 'SBX_GW_IP|HTTPS_PROXY'
→ HTTPS_PROXY=http://10.212.14.2:3128          (no SBX_GW_IP — guard unexempted)
getent hosts nonexistent-zzz-12345.example
→ 10.212.14.2                                  (catch-all: a name that cannot
                                                exist still resolves)
```

Three facts together close it: the server is pid 1, it has no `SBX_GW_IP`, and
DNS answers everything with the gateway address — so `resolveAndCheckHostname`
sees a `10.0.0.0/8` address for every discovery URL and throws before any
request is sent. `curl` works because it goes through `HTTPS_PROXY` and never
consults the guard.

Run the same two commands to classify any instance before choosing a lane
(§9.0).

**The reported error is misleading.** Both discovery catch blocks discard the
underlying exception (`catch (e) { /* try the next candidate */ }`), so an SSRF
block, a DNS failure, and a genuine 404 are indistinguishable — all read as
"no protected-resource metadata". Do not trust that string as a diagnosis.
`[PAVE source — mcp-oauth-flow.js:210, 250]`

**Do not chase the 429.** The host-only PRM URL
`/.well-known/oauth-protected-resource` answers `429` while the path-suffixed
`/.well-known/oauth-protected-resource/mcp` answers `200`. Discovery builds the
suffixed candidate from the resource path and tries it first, and `GET /mcp`
names it explicitly in `WWW-Authenticate`, so this is **not** the failure cause.
`[verified 2026-11-06 — GET /mcp → 401 + resource_metadata=<suffixed url>; suffixed 200, host-only 429, deterministic both orders]`

Fix options, in the order to try them:

1. Put `SBX_GW_IP` in the server process env. This is the only fix that
   repairs the native lane, but it needs the sbx driver — a container restart,
   not something editable from inside the agent sandbox. Try it first because
   everything else is a workaround.
2. Pin `authorizeUrl` + `tokenUrl` on the entry so discovery is skipped
   entirely (Outcome 2), remembering that pins make `clientId` mandatory.
3. Use the no-redirect manual lane (Outcome 4) — needs neither discovery nor a
   reachable callback, so it also survives a broken `PAVE_PUBLIC_BASE_URL`.

## 9. Setup runbook — zero to working

Follow this top to bottom. §9.0 decides which lane you are on; getting that
wrong is what costs the most time, because the native lane fails with an error
that points at the wrong subsystem (§8).

### 9.0 Preflight — two commands, decides everything

```bash
tr '\0' '\n' < /proc/1/environ | grep -E 'SBX_GW_IP|HTTPS_PROXY'
getent hosts nonexistent-zzz-12345.example
```

| Result | Lane | Why |
|---|---|---|
| `SBX_GW_IP` present | Native (Outcome 1) | The guard exempts the gateway, so discovery can complete |
| No `SBX_GW_IP` AND the fake name resolves | **Manual (Outcome 4)** | Catch-all DNS → guard blocks every discovery fetch. `oauth/start` can only fail |
| Neither | Native, then Outcome 2 | Normal instance; diagnose if discovery still fails |

If the fake name resolves, **do not attempt `oauth/start`**. It cannot succeed,
and its error message names the wrong subsystem. Go straight to Outcome 4.
`[verified 2026-11-06]`

### 9.1 Install the doctrine

```bash
pave install johnmak101-ops/openpave-dropbox-mcp
cd ~/.pave/skills/dropbox-mcp && git pull --ff-only
```

`pave install` can land a stale clone — pull before reading the doctrine, or you
will follow a version that lacks this runbook. Confirm:
`grep -c 'Outcome 4' SKILL.md` must be >= 1.
`[verified 2026-11-06 — install landed 0.1.1; a pull was required to pick up the 163-line update]`

### 9.2 Register the server

```bash
capabilities_add https://mcp.dropbox.com/mcp
```

Expect `status: error`. That is normal pre-auth, not a failure. **Never run this
command twice** — it rewrites `auth` to `none` and destroys the config. The
repair verb is always `capabilities_refresh dropbox`. `[verified 2026-11-06]`

### 9.3 Native lane (only when 9.0 said `SBX_GW_IP` present)

1. Back up `~/.pave/mcp/servers.json`, then set `dropbox.auth.type = "oauth2"`.
2. `POST /capabilities/mcp/dropbox/oauth/start` with `x-pave-internal-token`
   and `x-pave-internal-user`.
3. Present the returned `authorizeUrl` per §9.5.
4. If the browser is outside the container, set `PAVE_PUBLIC_BASE_URL` first —
   otherwise the consent redirect cannot reach `/oauth/callback` (§2).

### 9.4 Manual lane (Outcome 4, condensed)

1. **User action** — create the app: Developer Console → **Full Dropbox** →
   the scopes in §2 → Settings → **Allow public clients** → **Redirect URIs
   empty**. Only the **App Key** is needed. The App Secret is never used in a
   public-client PKCE flow: do not ask for it, do not store it, and if the user
   pastes it anyway, discard it and say so. `[verified 2026-11-06]`
2. Build the authorize URL (§2 Outcome 4 step 2) and present it per §9.5.
3. User consents; Dropbox shows the code on the page; user pastes it back.
4. Exchange with **both** `client_id` and `code_verifier` (§2 step 5).
5. `PUT /capabilities/mcp/dropbox/tokens` with `{"access","refresh"}`.
6. Write the refresh pins to `servers.json` (§2 step 7) — read them back and
   compare verbatim; a typo here silently disables refresh.
7. `capabilities_refresh dropbox`.

### 9.5 Presenting the authorize URL — do not use a question card

Emit the URL as a **clickable markdown link in the chat reply**:

```
[Authorize Dropbox](https://www.dropbox.com/oauth2/authorize?client_id=...)
```

Question-card text is not copyable in the WebUI. An OAuth URL the user cannot
copy is a dead end, and the whole lane stalls on it.
`[verified 2026-11-06 — user-reported improvement: "Question-card text is not
copyable for this user — emit long URLs (e.g. OAuth authorize links) as
clickable markdown links in the chat reply instead"]`

### 9.6 Grant and verify

```yaml
# ~/.pave/permissions.yaml
permissions:
  mcp:
    allowed:
      - mcp:dropbox
```

Then `tool_search "dropbox"` → **45** tools, `who_am_i`, `get_usage_and_quota`.
`[verified 2026-11-06]`

### 9.7 Clean up

Delete every temp file that held a token, a code, or a PKCE verifier — e.g.
`/tmp/dbx-exchange.json`, authorize-URL files, verifier files. Keep
`tokens.yaml` at mode 0600. `[verified 2026-11-06]`

### 9.8 Known-good state — what "done" looks like

| Artifact | Expected value |
|---|---|
| `servers.json → dropbox.status` | `installed` |
| `servers.json → dropbox.auth` | `type: "oauth2"` + `oauth.{clientId, tokenEndpoint, authorizeUrl, tokenUrl}` |
| `tokens.yaml` | `MCP__dropbox__ACCESS` + `MCP__dropbox__REFRESH`, mode 0600 |
| `permissions.yaml` | `permissions.mcp.allowed` contains `mcp:dropbox` |
| `tool_search "dropbox"` | 45 `mcp_dropbox_*` tools |
| `who_am_i` | returns account + team |

Compare against this table rather than against "it seemed to work". Every row
was observed on a working install. `[verified 2026-11-06]`

### 9.9 After ~4 h — confirm unattended refresh

The access token lives ~14400 s. Setup is not fully proven until a call succeeds
**after** that expiry without human help. When the window has passed:

1. Call `who_am_i`.
2. Success → unattended refresh works; the install is complete.
3. `401` → run `capabilities_refresh dropbox` once, then re-check. If that
   fixes it, refresh is not unattended and §2 step 7's pins need re-checking
   (compare them verbatim against `servers.json`).

Whatever the outcome: **never** run `capabilities_add` to recover — it resets
`auth` to `none` and destroys the config.

`[untested — the working run installed at ~04:05Z and scheduled this check for
~08:05Z rather than waiting; the result was not recorded]`

