# Headless subscription auth for Claude Code and Codex on Linux

Research for issue #4. Researched 2026-10-12 against Claude Code 2.1.296 and codex-cli 0.162.1 (local `--help`), the official docs, and `openai/codex` at commit `a9b0815167c27ab13719dd12d13b63e02b70d221`.

Each claim is tagged **[verified]** (stated by the cited primary source) or **[inferred]** (our reading of source code or docs, not stated outright, or not tested on a real host).

## TL;DR

| | Claude Code | Codex |
|---|---|---|
| Headless login | `claude auth login` prints a URL. You open it on any browser, sign in, and paste the login code shown there back into stdin. This is a paste-back authorization code, not an RFC 8628 device-code flow. | `codex login --device-auth` prints a link and a one-time code. You enter the code in any browser. Device-code login is in beta and must first be enabled in ChatGPT security settings. |
| Cron-friendlier option | `claude setup-token` prints a one-year subscription OAuth token. Put it in `CLAUDE_CODE_OAUTH_TOKEN`. It is never refreshed. | None for personal plans. Rely on `auth.json`, which refreshes itself. |
| Credential location | `~/.claude/.credentials.json` (mode 0600), or `$CLAUDE_CONFIG_DIR/.credentials.json` | `$CODEX_HOME/auth.json` (default `~/.codex/auth.json`), plaintext, depending on `cli_auth_credentials_store` |
| Self-refresh when unattended | Yes, for a `/login` credential. The token from `setup-token` does not refresh and lasts one year. | Yes. It refreshes when `last_refresh` is older than about 8 days, or after a 401, and writes the result back to `auth.json`. |
| Failure in non-interactive mode | `claude -p` exits non-zero and prints the error as the result on stdout (`Failed to authenticate: OAuth session expired and could not be refreshed`, error code `authentication_failed`). `claude auth status` exits 0 when logged in and 1 when not. | `codex exec` exits 1 on an error event [inferred]. `codex login status` exits 0 or 1 but only checks that a credential is stored, not whether it still works. |
| An API key in the env overrides the subscription? | **Yes.** In `-p` mode, `ANTHROPIC_API_KEY` is always used when present. `ANTHROPIC_AUTH_TOKEN` also overrides. | **`CODEX_API_KEY`: yes**, in `codex exec`. **`OPENAI_API_KEY`: no**, it is not read for model auth [inferred from source]. |

## Claude Code

### Login commands

- **[verified]** `claude auth login [--claudeai|--console|--sso|--email]`. `--claudeai` (the Claude subscription) is the default. Source: local `claude auth login --help` (2.1.296) and https://code.claude.com/docs/en/cli-reference
- **[verified]** On SSH or container hosts, the browser cannot reach Claude Code's local callback server. After you sign in, the browser shows a login code instead of redirecting back, and you paste it at `Paste code here if prompted`. `claude auth login` "reads the pasted code from standard input" and prints the URL so you can open it on another machine. Source: https://code.claude.com/docs/en/troubleshoot-install (section "OAuth login fails in WSL2, SSH, or containers") and https://code.claude.com/docs/en/authentication
- **[verified]** `claude setup-token` runs the same browser authorization flow but only prints a **one-year** OAuth token. It does not save the token anywhere. You set it as `CLAUDE_CODE_OAUTH_TOKEN`. It requires a Pro, Max, Team, or Enterprise plan and "can only make model requests". `--bare` mode does not read it. Source: https://code.claude.com/docs/en/authentication#generate-a-long-lived-token
- **[verified]** `claude auth status` prints JSON by default (`--text` for human-readable output) and exits **0 if logged in, 1 if not**. `authMethod` is one of `none`, `claude.ai`, `oauth_token`, `api_key`, `api_key_helper`, or `third_party`. Source: https://code.claude.com/docs/en/cli-reference
- **[inferred]** Claude Code has no RFC 8628 device-code flow (no code entered at a vendor URL). The "login code" goes the other way: the browser shows it and you paste it into the host. In practice this meets the "no browser on the host" need.

### Where credentials live

- **[verified]** On Linux, credentials are stored in `~/.claude/.credentials.json` with mode `0600`. If `CLAUDE_CONFIG_DIR` is set, the file lives under that directory. Source: https://code.claude.com/docs/en/authentication#credential-management
- **[inferred]** Cron runs with a minimal environment, so the job must run as the same user (same `$HOME`), or with the same `CLAUDE_CONFIG_DIR` that was used at login.

### Lifetime and refresh

- **[verified]** A `/login` credential is renewed automatically. Docs describe "Claude Code tried to renew your saved claude.ai login" and say parallel sessions "coordinate its renewal so that only one process refreshes the token at a time". Sources: https://code.claude.com/docs/en/errors#login-expired and https://code.claude.com/docs/en/troubleshoot-install#not-logged-in-or-token-expired
- **[verified]** The login does have an end date. When it is within three days of expiring, startup shows `Your login expires in 3 days · run /login to renew`. The warning only appears when a claude.ai login is the active credential. Source: https://code.claude.com/docs/en/authentication#renew-an-expiring-login
- **[verified]** Token validation depends on an accurate system clock. Source: troubleshoot-install, as above.
- **[verified]** A `CLAUDE_CODE_OAUTH_TOKEN` token lasts one year. Sessions using it "don't use the saved login and never see" `Login expired`. When it expires or is revoked, Claude Code keeps sending it and the request fails with 401. Source: https://code.claude.com/docs/en/errors#oauth-token-revoked-or-expired
- **Unknown:** the absolute lifetime of a `/login` refresh token. The docs give no number. We also don't know whether a single `claude -p` run every few hours is enough to keep it alive indefinitely. The 3-day expiry warning suggests a hard expiry that a refresh may not extend.
- **Unknown:** whether a cron-only (`-p`) user would ever see the 3-day warning. It is described as a startup message.

### What happens on expiry (non-interactive)

- **[verified]** `claude -p` "exits with code 0 on success and a non-zero code when the run fails ... When a failure happens inside the run, such as missing authentication, Claude Code prints the failure as the result on stdout." Source: https://code.claude.com/docs/en/headless
- **[verified]** If the refresh token is rejected, Claude Code clears the saved credentials. After that, each request stops locally. In `-p` mode the message is `Failed to authenticate: OAuth session expired and could not be refreshed`, with structured error code `authentication_failed`. Source: https://code.claude.com/docs/en/errors#login-expired
- **[verified]** For a revoked or expired token rejected by the API, `-p` prints `Failed to authenticate: OAuth token revoked. Please log in again or contact your administrator.` or `Failed to authenticate. API Error: 401 OAuth token has expired ...`. Source: https://code.claude.com/docs/en/errors#oauth-token-revoked-or-expired
- **[verified]** If another process holds the shared refresh lock, the request stops with `Could not refresh your login because another Claude Code process is refreshing it`. This is not a rejection of the login. Source: https://code.claude.com/docs/en/errors#could-not-refresh-your-login
- **[verified]** The result JSON (`--output-format json`) has an `error` field. One of its values is `authentication_failed`. Source: https://code.claude.com/docs/en/cli-reference
- **Unknown:** the exact non-zero exit code for an auth failure. The docs only say "non-zero".

### Does `ANTHROPIC_API_KEY` override the subscription?

- **[verified]** Yes. The precedence order is: cloud provider env, then `ANTHROPIC_AUTH_TOKEN`, then `ANTHROPIC_API_KEY`, then `apiKeyHelper`, then `CLAUDE_CODE_OAUTH_TOKEN`, then Anthropic profile or federation, then `/login` subscription OAuth. "In non-interactive mode (`-p`), the key is always used when present". Interactive mode asks once for approval first. Source: https://code.claude.com/docs/en/authentication#authentication-precedence
- **[inferred]** The warm-up script should `unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN` and the `CLAUDE_CODE_USE_*` variables, and should not pass `--bare`. It can assert `claude auth status` reports `authMethod` = `claude.ai` (or `oauth_token`) before sending.

## Codex

### Login commands

- **[verified]** `codex login` (ChatGPT sign-in) is the default. `codex login --device-auth` runs the device-code flow: "Open the link, sign in, and enter the one-time code." Device-code login is **beta and must be enabled in your ChatGPT security settings** (personal accounts) or workspace permissions (admin). The docs recommend it first for headless hosts. Source: https://developers.openai.com/codex/auth (now redirects to https://learn.chatgpt.com/docs/auth)
- **[verified]** Documented fallbacks: run `codex login` on a machine with a browser and copy `~/.codex/auth.json` to the headless host, or forward the callback port with `ssh -L 1455:localhost:1455`. Source: same page.
- **[verified]** If the browser does not open, `codex login` itself prints a hint: "On a remote or headless machine? Use `codex login --device-auth` instead." The CLI also has a path that prefers device code when it detects a headless environment and falls back to browser login if device code is unsupported. Source: https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/cli/src/login.rs
- **[verified]** `codex login status` prints `Logged in using ChatGPT` (or the API key or token variant) and exits 0. It prints `Not logged in` and exits 1 otherwise. Source: same file (`run_login_status`).
- **[inferred]** `codex login status` loads the stored credential without contacting the server, so it reports "Logged in" even when the refresh token is dead. It is not a health check.

### Where credentials live

- **[verified]** `cli_auth_credentials_store` sets where credentials go. `file` means `auth.json` under `CODEX_HOME` (default `~/.codex`). `keyring` means the OS credential store. `auto` means keyring if available, otherwise file. `ephemeral` means in-memory only. The file is plaintext: "treat `auth.json` like a password." Source: https://learn.chatgpt.com/docs/auth
- **[inferred]** On a headless Linux host with no Secret Service or D-Bus session, `auto` will fall back to `auth.json`. Setting `cli_auth_credentials_store = "file"` explicitly avoids surprises under cron.

### Lifetime and refresh

- **[verified]** Codex "refreshes tokens automatically during use before they expire". Source: https://learn.chatgpt.com/docs/auth
- **[verified]** If `last_refresh` is older than about 8 days, Codex refreshes the token bundle before the run continues and writes the new tokens and `last_refresh` back to `auth.json`. A 401 triggers refresh-and-retry. One `auth.json` per runner. Do not share one across machines or concurrent jobs, because rotation by another holder breaks it. The flow "does not guarantee the same session lasts forever." Source: https://developers.openai.com/codex/auth/ci-cd-auth (now at https://learn.chatgpt.com/docs/auth/ci-cd-auth)
- **[verified]** In source, `TOKEN_REFRESH_INTERVAL = 8` (days), and the access token is refreshed 5 minutes before it expires (`CHATGPT_ACCESS_TOKEN_REFRESH_WINDOW_MINUTES = 5`). Refresh goes to `https://auth.openai.com/oauth/token`. Source: https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/login/src/auth/manager.rs
- **[verified]** Source distinguishes these refresh failures: `refresh_token_expired`, `refresh_token_reused`, and `refresh_token_invalidated`. Each maps to a message ending "Please log out and sign in again." Example: "Your access token could not be refreshed because your refresh token has expired." Source: same file.
- **[inferred]** `refresh_token_reused` means refresh tokens rotate and are single-use. Running two Codex hosts or concurrent jobs from one copied `auth.json` will break one of them.
- **Unknown:** the absolute refresh-token lifetime for ChatGPT sign-in. The docs only give the roughly 8-day staleness threshold. A warm-up every few hours should stay well inside it.

### What happens on expiry (non-interactive)

- **[inferred]** `codex exec` sets `error_seen` when the server emits an error notification and then calls `std::process::exit(1)`. A refresh failure should therefore surface as one of the messages above (on stderr or in `--json` events) with exit code 1. Source: https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/exec/src/lib.rs. Not tested end to end.
- **[verified]** Docs say only that Codex "may start returning 401s that it can't recover from". The fix is to run `codex login` again on a trusted machine. Source: ci-cd-auth page.
- **[verified]** If `forced_login_method` (`"chatgpt"` or `"api"`) is set and the active credentials don't match, "Codex logs the user out and exits." Source: https://learn.chatgpt.com/docs/auth

### Does an API key override ChatGPT login?

- **[verified]** `CODEX_API_KEY` takes priority over everything else when the caller enables it ("API key via env var takes precedence over any other auth method"). `codex exec` enables it (`enable_codex_api_key_env: true`). Docs confirm `CODEX_API_KEY` works with `codex exec`, `codex review`, and the SDK. Sources: `manager.rs` `load_auth()` and `exec/src/lib.rs` at the commit above, and https://learn.chatgpt.com/docs/non-interactive-mode
- **[verified]** The next sources in order are external ChatGPT tokens held in memory, then `CODEX_ACCESS_TOKEN` (an enterprise access token), then the persisted `auth.json` or keyring. Source: `manager.rs` `load_auth()`.
- **[inferred]** `OPENAI_API_KEY` is **not** read by `load_auth()`. In source, `read_openai_api_key_from_env()` is used only to prefill TUI onboarding and for realtime conversation. So it alone will not displace a ChatGPT login for `codex exec`. A custom `model_provider` with `env_key = "OPENAI_API_KEY"` would change this.
- **[inferred]** Setting `forced_login_method = "chatgpt"` in `~/.codex/config.toml` keeps an env API key from being used, because `load_auth` skips API-key auth when it is not allowed. It also protects against a stray `CODEX_API_KEY`. The script should `unset CODEX_API_KEY CODEX_ACCESS_TOKEN` anyway.

## Implications for cc-warmup

- **[inferred]** After a one-time interactive login, both CLIs can run unattended from cron on subscription auth. The warm-up itself keeps Codex's `auth.json` fresh, because runs happen far more often than every 8 days. The same is probably true for Claude's `/login` credential.
- **[inferred]** For Claude, `claude setup-token` plus `CLAUDE_CODE_OAUTH_TOKEN` is the documented automation path. It is still subscription auth with a predictable expiry (one year), and it avoids refresh-lock contention with any interactive `claude` the user runs on the same host. The trade-off is a yearly manual renewal and a secret stored in the cron environment.
- **[inferred]** Pre-flight checks: `claude auth status` (exit code plus `authMethod`) and `codex login status`. Neither proves the token still refreshes, so the script must also treat a non-zero exit from the warm-up call as "needs re-login" and alert.
- **[inferred]** Sanitize the environment before each call: unset `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`, `CODEX_API_KEY`, and `CODEX_ACCESS_TOKEN`. Pin `HOME`, `CLAUDE_CONFIG_DIR`, and `CODEX_HOME`.

## Open questions

1. What is the absolute lifetime of a Claude `/login` refresh token? Does periodic `-p` use extend it, or does it hard-expire, requiring a manual re-login (with the 3-day warning)?
2. Should cc-warmup use `CLAUDE_CODE_OAUTH_TOKEN` (one-year, no refresh) or the saved `/login` credential? This affects how the token is stored in the cron environment.
3. What exact exit codes and output does each CLI produce on an auth failure? Claude documents only "non-zero", and Codex's behavior is inferred from source. This needs a real test, for example with a corrupted copy of the credentials file.
4. Device-code login for Codex is beta and requires a ChatGPT security setting. Can the user enable it on their plan, or is the fallback (copying `auth.json` or SSH port forwarding) needed?
5. If the user also runs Claude Code or Codex interactively on the same host and account, could a warm-up's token refresh conflict with an interactive session? Claude coordinates refresh with a lock, and Codex warns against sharing `auth.json`.
