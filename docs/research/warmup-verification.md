# Can a warm-up's success be verified from the CLI?

Research for issue #5. Researched 2026-10-12 against Claude Code 2.1.296 and codex-cli 0.162.1, with openai/codex at commit `a9b0815` (2026-10-11).

## Answer

**Yes, for both providers, and without spending quota.** Each CLI has a non-interactive way to read the current usage window: how much is used and when it resets. A warm-up can be checked in three layers:

1. **The process succeeded.** Exit code 0, plus a success marker in the structured output.
2. **A real model turn happened.** The structured output reports non-zero token usage.
3. **A usage window is running.** Read the 5-hour window's reset time after the warm-up. If it lies about 5 hours after the warm-up was sent, the warm-up started the window. The read itself sends no prompt to the model.

| | Claude Code | Codex |
|---|---|---|
| Exit code | 0 on success, non-zero on failure | 1 when the turn fails or errors |
| Structured output of the warm-up | `-p --output-format json`: `is_error`, `subtype`, `usage`, `num_turns` | `exec --json`: `turn.completed` (with `usage`) or `turn.failed` / `error` |
| Window state in the warm-up's own output | Not in `json`. Possibly a `rate_limit_event` in `stream-json` (unconfirmed, see below) | No. The JSONL stream carries no rate-limit event |
| Read window state without a prompt | `claude -p "/usage" --output-format stream-json --verbose` | `codex app-server` JSON-RPC `account/rateLimits/read` |
| Window state as a by-product | (none found) | `token_count` events in `~/.codex/sessions/**/rollout-*.jsonl` |

## Verified facts

### Claude Code

- **Exit codes.** "Claude Code exits with code 0 on success and a non-zero code when the run fails ... When a failure happens inside the run, such as missing authentication, Claude Code prints the failure as the result on stdout." ([headless docs](https://code.claude.com/docs/en/headless))
- **JSON output.** `--output-format json` returns "structured JSON with result, session ID, and metadata", including `total_cost_usd` and usage. ([headless docs](https://code.claude.com/docs/en/headless)) Observed fields on the result object include `is_error`, `subtype` (`"success"`), `num_turns`, `usage.{input_tokens,output_tokens,...}`, `modelUsage`, `total_cost_usd`, and `session_id`.
- **Retry and error events.** `stream-json` emits `system/api_retry` events whose `error` field can be `rate_limit`, `authentication_failed`, `billing_error`, and so on. ([headless docs](https://code.claude.com/docs/en/headless#handle-api-retries))
- **`/usage` works in print mode and is structured.** "When you send `/usage` as a prompt, Claude Code delivers the report as an assistant message ... It attaches `usage_report` [`SDKUsageReport`] to that same message" when the session uses a claude.ai credential that has a known plan type or the `user:profile` scope, and the account isn't on usage-based billing. A `claude setup-token` token (`CLAUDE_CODE_OAUTH_TOKEN`) "doesn't qualify by default, because it carries only the `user:inference` scope." The type is marked experimental. ([Agent SDK TypeScript reference, SDKAssistantMessage / SDKUsageReport](https://code.claude.com/docs/en/agent-sdk/typescript#sdkusagereport))
- `SDKUsageReport.rate_limits.limits[]` rows carry `kind` (`session`, `weekly_all`, `weekly_scoped`, ...), `percent` (0-100), and `resets_at` (an ISO 8601 string, or null). The docs say to classify rows on `kind`. (same source)
- **Observed locally** (Pro plan): `claude -p "/usage" --output-format stream-json --verbose --no-session-persistence` exited 0. The assistant message had `model: "<synthetic>"` and zero tokens, and the result had `num_turns: 0` and `total_cost_usd: 0`. So no model call was made. The `usage_report` contained:
  ```json
  {"kind":"session","group":"session","percent":15,"resets_at":"2026-10-12T19:49:59.893875+09:00","is_active":true, ...}
  {"kind":"weekly_all","group":"weekly","percent":2,"resets_at":"2026-10-17T04:59:59.8939+09:00", ...}
  ```
  With `--output-format json`, only the text report appears, in `result` (for example `Current session: 15% used · resets Oct 12, 7:49pm`). To get the structured field, use `stream-json`.
- **Rate-limit event type exists.** The SDK defines `SDKRateLimitEvent {type:"rate_limit_event", rate_limit_info:{status:"allowed"|"allowed_warning"|"rejected", resetsAt?, utilization?, ...}}`, documented as "Emitted when the session encounters a rate limit." ([SDK reference](https://code.claude.com/docs/en/agent-sdk/typescript#sdkratelimitevent))
- **Status-line JSON** has `rate_limits.five_hour.{used_percentage,resets_at}` for Pro/Max subscribers, "only after the first API response in the session". This only exists in the interactive TUI, so it doesn't help a cron job. ([statusline docs](https://code.claude.com/docs/en/statusline))
- **Pre-flight auth check.** `claude auth status` prints JSON with `loggedIn`, `authMethod` (`"claude.ai"`), and `subscriptionType`. (local `claude auth status`)

### Codex

- **JSONL events.** "`stdout` becomes a JSON Lines (JSONL) stream ... Event types include `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.*`, and `error`." `turn.completed` carries `usage` (input, cached, and output tokens). ([non-interactive docs](https://developers.openai.com/codex/noninteractive))
- The full `ThreadEvent` enum in source has only those variants. There is **no rate-limit event** in `codex exec --json`. ([`codex-rs/exec/src/exec_events.rs`](https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/exec/src/exec_events.rs))
- **Exit code.** `codex exec` sets `error_seen` on a non-retried `Error` notification, or when the turn completes as `Failed` or `Interrupted`, and then calls `std::process::exit(1)`. ([`codex-rs/exec/src/lib.rs` L1338-1403](https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/exec/src/lib.rs#L1338-L1403))
- **Rate-limit snapshot type.** `TokenCountEvent { info, rate_limits: Option<RateLimitSnapshot> }`. The snapshot has `primary` and `secondary` windows, each `{used_percent, window_minutes, resets_at (unix s)}`, plus `plan_type` and `rate_limit_reached_type`. ([`codex-rs/protocol/src/protocol.rs` L2464-2523](https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/protocol/src/protocol.rs#L2464-L2523))
- **Prompt-free read.** The app-server protocol defines the request `account/rateLimits/read` and the notification `account/rateLimits/updated`. ([`codex-rs/app-server-protocol/src/protocol/common.rs`](https://github.com/openai/codex/blob/a9b0815167c27ab13719dd12d13b63e02b70d221/codex-rs/app-server-protocol/src/protocol/common.rs)) `codex app-server` is labelled `[experimental]` in `codex --help` and defaults to stdio transport.
- **Observed locally.** Sending `initialize`, then `initialized`, then `{"method":"account/rateLimits/read","id":1}` to `codex app-server` over stdio returned the following, with no prompt sent:
  ```json
  {"rateLimits":{"limitId":"codex","primary":{"usedPercent":15,"windowDurationMins":10080,"resetsAt":1792375429},"secondary":null,"planType":"self_serve_business_prolite",...}}
  ```
- **Observed locally: session rollouts** (`~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`) record `event_msg` / `token_count` lines with the same `rate_limits` snapshot after turns. On a `plus` plan the snapshot was `primary:{window_minutes:300,...}` (5 h) and `secondary:{window_minutes:10080,...}` (weekly). So a script can read window state from the rollout file that `codex exec` writes, without making a second call. (This doesn't apply with `--ephemeral`, which doesn't persist the session.)
- **Pre-flight auth check.** `codex login status` prints `Logged in using ChatGPT` and exits 0. (local)

## Inferences (not verified)

- **Verification rule.** After a warm-up at time T, the 5-hour row's reset time should be about T + 5 h (Claude `kind:"session"`; Codex the window with `window_minutes == 300`). If the reset time is earlier, a window was already running and the warm-up did not start a new one. Reading the window before the warm-up shows whether it's already running.
- **Reads don't start windows.** Both reads make no model call (observed: zero tokens and `<synthetic>` model on Claude, and a plain account read on Codex). So they presumably neither consume quota nor start a window. We haven't tested this directly by reading while no window is running.
- **Claude `rate_limit_event`.** The `rate_limit_event` may appear in a normal `stream-json` warm-up with `status:"allowed"` and a `resetsAt`, which would make the separate `/usage` read unnecessary. The docs only say it is emitted "when the session encounters a rate limit". We didn't test this, because testing it needs a real prompt.

## Unknowns and caveats

- **Plan-dependent windows.** The Codex account observed today (`self_serve_business_prolite`) reports only a weekly primary window (`10080` min, `secondary: null`). No 5-hour window was reported, even though a `plus` login on the same machine yesterday did report one. **Some plans may have no 5-hour window to warm up.** Verification has to look the window up by `window_minutes == 300` rather than assuming `primary`. Whether a 5-hour row is absent while idle or absent for this plan entirely is unknown.
- **Idle shape.** We don't know what each read returns when no window is running: a Claude `session` row with `resets_at: null` or 0%, or no row at all, and a Codex window that is null or 0%.
- **Stability.** `SDKUsageReport` is "experimental: its shape may change", and `codex app-server` is `[experimental]`. Both need defensive parsing. The `/usage` text form is the fallback for Claude.
- **Long-lived tokens.** Headless Claude auth through a `setup-token` token doesn't produce `usage_report` by default (`user:inference` scope only). On a cron host, the login method decides whether structured verification is available.
- **Authority.** These reads come from the client. The provider's own backend is the authority, and nothing here proves server-side accounting beyond what the backend returns.
