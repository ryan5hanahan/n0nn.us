---
title: A security review of oh-my-pi
---
I ran a code review across the oh-my-pi coding agent, hunting for hidden tracking, backdoor-shaped code paths, and misconfigured hardening. What follows is the consolidated, severity-ranked result. The work was split across several focused passes (telemetry, remote-control, tool sandbox, network listeners, supply chain) and every top claim was verified against source.

**Bottom line:** no deliberate backdoor, hardcoded live credential, identity special-casing, or covert analytics SDK was found. The real exposure is architectural. Repository content is fully trusted, tool approval defaults to `yolo`, and one default-on reporting feature ships a persistent install UUID under a consent prompt that never names where the data goes.

## Critical

**1. Opening a repo executes its code, with no trust prompt.** `isProjectTrusted()` in `extensibility/extensions/types.ts` always returns true by design. Hooks in `.omp/hooks/*` and `.claude/hooks/*` are live-imported, as are `.omp/extensions`, `.omp/commands`, and `.omp/tools`. Project `.mcp.json` stdio servers are spawned because `mcp.enableProjectConfig` defaults true. The trigger is simply `git clone; omp`. Only `--no-extensions` or an explicit `--trusted-extension` allowlist gates it.

**2. Project `.env` loads into the omp process with no key denylist** (`packages/utils/src/env.ts`). Repo-controlled keys that matter:

- `PI_SHELL_PREFIX` is prepended to every bash tool command, so it runs arbitrary code on the first tool call.
- `PI_CONFIG_FILES` injects settings overlays that sit inside the operator-authority view, defeating the one protection that exists for `bash.patterns`.
- `OMP_AUTH_BROKER_URL` plus `OMP_AUTH_BROKER_TOKEN` redirect `/login` credential uploads to an attacker host. The docs claim project values cannot do this; that is true for config files and false for `.env`.
- `PI_BUILD_URL` redirects self-update, including its checksum.
- `PI_CODING_AGENT_DIR` repoints the entire agent config directory.
- `PI_AUTO_QA_PUSH=1` with `PI_AUTO_QA_PUSH_URL` silently ships model-authored reports to a chosen URL.

## High

**3. Project settings layer has no per-key scope.** `config/settings.ts` deep-merges every key from `.omp/settings.json`, `.omp/config.yml`, and `.claude/settings.json`. Only `bash.patterns` is read through the operator-authority view. A repo can therefore set `tools.approvalMode: yolo`, `tools.approval.bash: allow`, `update.auto`, `marketplace.autoUpdate: auto`, `skills.registryUrl`, and extra `extensions` paths.

**4. Approval mode defaults to `yolo`** (`tools/settings.ts`). Documented, but it turns every injection vector below into code execution with no click. Subagents are forced to `yolo` regardless.

**5. The web fetch tool has no SSRF guard.** `tools/fetch.ts` accepts any http(s) URL, follows redirects, and nothing in the web, markit, or utils layers filters loopback, link-local, or RFC1918. Reachable targets include cloud metadata at `169.254.169.254`, the local stats dashboard (full transcripts of other sessions), and a local auth-gateway running `--no-auth`.

**6. Model-triggerable unverified binary installs.** `utils/tools-manager.ts` fetches `releases/latest` of yt-dlp, sd, and ast-grep, then chmods and executes, with no hash, pin, or signature. `trafilatura` is pip-installed unpinned. A YouTube URL or a fetch fallback triggers it. The correct pinned-SHA pattern already exists in `utils/grammars.ts`.

## Medium

**7. Auto QA "grievance" push to `qa.omp.sh`.** `dev.autoqa` defaults true, and the system prompt instructs the model to file reports. The payload carries a persistent UUID from `~/.omp/install-id`, platform, arch, model, and LLM-written free text. Redaction covers only the home path and username. The consent dialog is a randomized joke ("Spill it to the devs? ... never anything personal") and names neither the endpoint nor the identifier. Headless runs default to no consent, which is correct. Disclosed in docs, but not in the setup wizard or README.

**8. The install UUID correlates the user across providers.** The same id reaches Anthropic as a derived `device_id`, OpenAI Codex as `installationId`, OpenCode as `x-opencode-session`, and `qa.omp.sh`. Each is provider-bound and mimics official-client metadata, but one identifier links all four.

**9. Local servers trust the network too much.** The auth-gateway sends `Access-Control-Allow-Origin: *` and allows `--no-auth` on `0.0.0.0`, so any website can bill the user's credentials. The stats dashboard does no Host check, so DNS rebinding can read session transcripts. Metaharness (dev-only, not shipped) binds `0.0.0.0` with no auth and spawns processes from the request body.

**10. Client impersonation and scanner-evading secrets.** The Anthropic provider presents itself as Claude Code: User-Agent, system identity block, billing header, tool prefix, and Claude-shaped metadata. Google OAuth client secrets for Gemini CLI and Antigravity ship base64-encoded; a sibling file's comment says they are "stored base64 so secret scanners stay quiet." Not a backdoor, but an account-suspension risk the user carries, deliberately hidden from tooling.

**11. The self-update trust chain is single-origin.** The binary URL and its sha256 both come from `build.stencil.so`, with no detached signature. `scripts/install.sh` is `curl | sh` with the same property. The startup update check is default-on but sends only version and platform.

**12. Collab guests with a write link can approve tool prompts**, racing the local user. Documented, end-to-end sealed, and off by default.

## Low

- The OAuth `/launch` endpoint 302s with `state` in the Location header, enabling a login-CSRF by a local process.
- The browser relay `/cdp` has no token, only an Origin reject, so any local process can drive the logged-in Chrome. DAP's `runInTerminal` spawns with no approval.
- `claude-trace` sets `NODE_TLS_REJECT_UNAUTHORIZED=0` in its child process. Puppeteer is patched for bot-detection stealth. Dockerfiles run as root. Three CI actions are tag-pinned, and a long-lived `NPM_TOKEN` sits beside OIDC publishing.

## Verified clean

No postinstall scripts. The lockfile is 100% registry with sha512. Cargo denies non-crates.io sources. CI has no `pull_request_target`. The OAuth callback is loopback-only with a state check and PKCE. SSH uses `accept-new`, not `no`. There are no silent writes to rc files, git hooks, `~/.ssh`, or cron. OTLP export and the share/collab/stream uploads are explicit and end-to-end sealed. No remote config or kill-switch fetch exists.

## Suggested fix order

1. A per-workspace trust prompt covering findings 1 through 3 (they share the project-discovery path).
2. Denylist security-relevant keys from project `.env`.
3. Route approval settings through the operator-authority view, as `bash.patterns` already is.
4. Add a private-address filter to the fetch tool.
5. Pin tool downloads to a hash, reusing the grammar-download pattern.
