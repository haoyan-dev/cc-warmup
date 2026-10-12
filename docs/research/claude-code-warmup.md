# Does a minimal warm-up start the Claude Code usage window?

Research for issue #2. Sources checked 2026-10-12 against Claude Code v2.1.296 (installed locally). "Verified" means stated in a first-party Anthropic source linked inline; "Inference" means our reading of those facts; "Unknown" means no first-party source answers it and it needs an empirical test.

## Short answer

- **Yes, very likely**: any model request sent through `claude -p` with a Pro/Max subscription login draws on the same 5-hour session allowance as claude.ai and every model, so a one-message Haiku warm-up should start the usage window. No current first-party page states in so many words that "the window starts at the first message"; confirm it once by experiment (see "Verify on the host").
- **Invocation** (subscription auth, no API key, no tools, no prompts):

  ```bash
  env -u ANTHROPIC_API_KEY -u ANTHROPIC_AUTH_TOKEN \
    claude -p "Reply with the single word OK." \
      --model haiku --effort low \
      --safe-mode --strict-mcp-config --tools "" \
      --permission-mode dontAsk --permission-prompts none \
      --max-turns 1 --no-session-persistence \
      --output-format json
  ```

  Do **not** use `--bare`: it ignores the subscription login and accepts only `ANTHROPIC_API_KEY` or `apiKeyHelper`.
- **Success/failure**: the exit code is 0 on success and non-zero on failure. Classify failures with the JSON fields `is_error`, `subtype` and `api_error_status`, not with the exit code. Anthropic doesn't document which non-zero code each failure gets.

## 1. How the 5-hour usage window is defined

Verified:

- Paid plans have "a session limit that resets every five hours" and "weekly limits that reset once every week". ([How do usage and length limits work?](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work))
- "Every plan has usage limits that reset on a rolling five-hour session window, and paid plans add weekly limits on top." ([Pricing FAQ](https://claude.com/pricing))
- The window is shared across surfaces: "usage of all different Claude product surfaces (claude.ai, Claude Code, Claude Desktop) counts towards the same usage limit". ([11647753](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work); also [Use Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan): "These limits are shared across Claude and Claude Code")
- "Usage counts against the session and weekly allowances at the same time." ([Claude Code errors: You've hit your session limit](https://code.claude.com/docs/en/errors#youve-hit-your-session-limit))
- The weekly limit "resets at a fixed time each week that is assigned to your account", whenever you start using Claude. ([What is the Max plan?](https://support.claude.com/en/articles/11049741-what-is-the-max-plan)) A warm-up can't move the weekly reset.
- Claude Code exposes the window's reset time as `rate_limits.five_hour.resets_at` (Unix epoch seconds) to status-line scripts, and only "after the first API response in the session", for Pro/Max subscribers. ([Status line](https://code.claude.com/docs/en/statusline))

Unknown / unverified:

- **What starts a window.** None of the current help-center pages ([11647753](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work), [11145838](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan), [11049741](https://support.claude.com/en/articles/11049741-what-is-the-max-plan), [9797557](https://support.claude.com/en/articles/9797557-usage-limit-best-practices), [17007452](https://support.claude.com/en/articles/17007452-what-is-a-limit-reset)) says when a window starts. An older Anthropic help article (`support.anthropic.com/en/articles/11014257`, "About Claude's Max plan usage") said that sessions start with your first message and reset 5 hours later. That URL now returns 404 and its wording could not be re-checked (web.archive.org was unreachable from this environment). GLOSSARY.md's definition ("begins with the first message sent after the previous window has expired") matches that older article and the word "rolling", but it rests on a source that has been removed.
- Whether the reset time is truncated to the hour or set to the exact minute. Only third-party reports discuss this; ignore them until we test it.

## 2. Does a Haiku message start the same window?

Verified:

- "The session and weekly limits are shared across all models, so switching models doesn't restore access." Separate Opus and Sonnet limits exist, each applying "only to requests to that model family". ([errors](https://code.claude.com/docs/en/errors#youve-hit-your-session-limit); same wording in [Manage costs](https://code.claude.com/docs/en/costs#when-a-developer-asks-about-a-limit))
- Model choice affects how fast usage is consumed. ([11647753](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work), [pricing FAQ](https://claude.com/pricing))
- The `haiku` alias resolves to Haiku 5.5 on the Anthropic API ("Use v2.1.293 or later with Haiku 5.5"), Haiku is included in every plan ([pricing](https://claude.com/pricing)), and "Haiku models are always available and can't be disabled". ([Model configuration](https://code.claude.com/docs/en/model-config))

Inference: a Haiku request counts against the one cross-model session allowance, so it should start that same window, and it is the cheapest model to send it with. There's no Haiku-specific session limit to worry about. No source spells this out, so the host test below should confirm it.

## 3. The headless invocation

Verified facts behind each flag:

| Flag / step | Why | Source |
| - | - | - |
| `-p` | Runs non-interactively, then exits. The workspace trust dialog is skipped under `-p`. | `claude --help`; [headless](https://code.claude.com/docs/en/headless) |
| **no `--bare`** | Bare mode never reads OAuth or `CLAUDE_CODE_OAUTH_TOKEN`, so it would need an API key. | [headless: bare mode](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode); [authentication](https://code.claude.com/docs/en/authentication#generate-a-long-lived-token) |
| unset `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN`, no `apiKeyHelper` | These take precedence over the subscription. In `-p`, "the key is always used when present". | [authentication precedence](https://code.claude.com/docs/en/authentication#authentication-precedence) |
| `--model haiku` | Uses the cheapest model. | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--effort low` | Haiku 5.5 can't turn thinking off, but effort controls how much it thinks. `low` is a supported level. | [model-config](https://code.claude.com/docs/en/model-config); [costs](https://code.claude.com/docs/en/costs#adjust-extended-thinking) |
| `--safe-mode` | Disables CLAUDE.md, skills, plugins, hooks, MCP servers and auto memory. "Authentication, model selection, built-in tools, and permissions work normally." This is the subscription-compatible alternative to `--bare`. | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--strict-mcp-config` (no `--mcp-config`) | Prevents any MCP servers from loading. This guards against managed MCP servers that `--safe-mode` still lets through. | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--tools ""` | Disables all built-in tools. `EndConversation` is removed "only when no MCP tools remain". | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--permission-mode dontAsk` + `--permission-prompts none` | Any call that would prompt is denied instead of waiting for an answer. This is a safety net, since no tools are available anyway. `--permission-prompts` needs v2.1.259+. | [headless](https://code.claude.com/docs/en/headless#turn-off-permission-prompts-in-unattended-runs) |
| `--max-turns 1` | Caps the agentic loop. It "Exits with an error when the limit is reached". | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--no-session-persistence` | Stops the run from leaving a transcript on disk. | [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| `--output-format json` | Prints one result object with `is_error`, `subtype`, `api_error_status`, `total_cost_usd`, `usage` and `modelUsage`. | [headless](https://code.claude.com/docs/en/headless#get-structured-output); [SDK `SDKResultMessage`](https://code.claude.com/docs/en/agent-sdk/typescript) |

Auth on an always-on Linux host (verified, [authentication](https://code.claude.com/docs/en/authentication)):

- **Option A**: run `/login` once interactively on the host. Credentials are saved in `~/.claude/.credentials.json` (mode 0600) and refresh automatically. Once the stored login expires and can't be refreshed, requests fail with `Login expired · Please run /login`. Claude Code warns 3 days before expiry, but only in interactive sessions.
- **Option B**: run `claude setup-token` and export the result as `CLAUDE_CODE_OAUTH_TOKEN` in the cron environment. This is a one-year OAuth token on the subscription. It "can only make model requests", which is all a warm-up needs. If it is rejected with a 401, Claude Code does not fall back to the stored login.
- Pre-flight check: `claude auth status` "Exits with code 0 if logged in, 1 if not". Its JSON `authMethod` is one of `none`, `claude.ai`, `oauth_token`, `api_key`, `api_key_helper` or `third_party`. The warm-up should refuse to run unless it is `claude.ai` or `oauth_token`, which guards the subscription-only rule. ([CLI reference](https://code.claude.com/docs/en/cli-reference))

Smallest token cost, inference: even with `--safe-mode` and no tools, Claude Code still sends its default system prompt. `--system-prompt "Reply OK."` would replace it and shrink input tokens further. A warm-up uses only a tiny share of a 5-hour allowance either way, so this is optional. We haven't verified whether replacing the system prompt changes how the request is classified or billed.

## 4. Exit codes and output

Verified:

- "Claude Code exits with code 0 on success and a non-zero code when the run fails." An invalid flag is reported on stderr before the run starts. "When a failure happens inside the run, such as missing authentication, Claude Code prints the failure as the result on stdout." ([headless](https://code.claude.com/docs/en/headless#basic-usage))
- SIGTERM gives exit 143. ([headless](https://code.claude.com/docs/en/headless#stop-a-run-with-sigterm)) Many startup and configuration errors exit with 1. ([errors](https://code.claude.com/docs/en/errors))
- The result JSON has `subtype` set to `success` or to `error_max_turns`, `error_during_execution`, `error_max_budget_usd` or `error_max_structured_output_retries`. It also carries `is_error: boolean` and `api_error_status`, "the HTTP status code of the API error that terminated the conversation", plus `errors: string[]` on the error branch. ([SDK reference](https://code.claude.com/docs/en/agent-sdk/typescript))
- Usage-limit message text: `You've hit your session limit · resets 3:45pm` (also weekly, Opus and Sonnet variants). With JSON output, these messages go in the `result` field. ([errors](https://code.claude.com/docs/en/errors#youve-hit-your-session-limit))
- With `--output-format stream-json`, `rate_limit_event` messages carry `rate_limit_info.status` (`allowed`, `allowed_warning` or `rejected`), `resetsAt` and `utilization`. `errorCode: "credits_required"` means the subscription's included usage is exhausted. ([SDK `SDKRateLimitEvent`](https://code.claude.com/docs/en/agent-sdk/typescript))
- Retryable failures emit `system/api_retry` events whose `error` field takes values including `authentication_failed`, `rate_limit`, `overloaded` and `billing_error`. ([headless: Handle API retries](https://code.claude.com/docs/en/headless#handle-api-retries))
- Authentication failure messages: `Not logged in · Please run /login`, `Login expired · Please run /login`, `Failed to authenticate: OAuth token revoked…` and `Failed to authenticate: OAuth session expired and could not be refreshed` (structured code `authentication_failed`), and `API Error: 401 Invalid authentication credentials`. ([errors](https://code.claude.com/docs/en/errors))
- A 429 caused by server-side throttling (`Server is temporarily limiting requests (not your usage limit)`) is told apart from a plan limit by whether the response carries the unified quota headers. ([errors](https://code.claude.com/docs/en/errors))

Unknown: the exact non-zero exit code for a usage-limit rejection or an auth failure under `-p`, and whether `api_error_status` is 429 or 401 in those cases or absent. Treat any non-zero exit as failure and log stdout and stderr.

## Verify on the host (open items)

1. **Window start.** When no window is active, run the warm-up with `--output-format stream-json --verbose` and record `rate_limit_event.rate_limit_info.resetsAt`. Check that it is about now + 5h and whether it is rounded to the hour. This is an inference: we expect a CLI stream-json run to emit `rate_limit_event`, because it is part of the SDK message union, but no doc confirms a successful `allowed` request emits one.
2. **Haiku starts the shared window.** After a Haiku warm-up, check claude.ai Settings > Usage, which should show the session as started with the matching reset time.
3. **Failure shapes.** Capture the exit code and JSON for a logged-out run (`CLAUDE_CODE_OAUTH_TOKEN=bogus`) to confirm auth-failure handling. Capture a usage-limited run when one happens naturally.
4. **Cost.** Read `modelUsage` from one run to record the actual token footprint, with and without `--system-prompt`.
