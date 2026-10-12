# Does a minimal warm-up start the Codex usage window?

Research for [issue #3](https://github.com/haoyan-dev/cc-warmup/issues/3). Researched 2026-10-12 against Codex CLI `0.162.1` (`rust-v0.162.1`, the latest release on that date).

Each claim is marked **Verified** (with its source) or **Inference / Unknown**.

## Short answer

- **Yes, probably.** OpenAI's help center says a new five-hour window starts with the first message sent in Work or Codex after the previous window ends. A one-message `codex exec` run is a Codex message, so it should start the window. No official source says that explicitly about `codex exec`, and nobody has observed it end-to-end yet.
- **Recommended warm-up:**

  ```sh
  codex exec --skip-git-repo-check --ephemeral --sandbox read-only \
    -m <cheapest-model> -c model_reasoning_effort="low" -c web_search="disabled" \
    --json "Reply with only the word OK. Do not run any commands." </dev/null
  ```

  Success means exit code `0` plus a `turn.completed` event. Any failure exits `1`. To tell a usage limit apart from an auth failure, read the `error`/`turn.failed` message text, or run `codex login status` first.
- **Pro plans currently have no five-hour limit**, so a warm-up does nothing useful for the 5-hour window on Pro.

## 1. How the limits are defined and when the window starts

| Claim | Status | Source |
|---|---|---|
| "Depending on your plan, usage limits may apply over a five-hour window and a weekly window. Where both apply, you need allowance remaining in both to continue." | Verified (help-center text, via search excerpt; direct fetch returned 403) | [help.openai.com 20001516](https://help.openai.com/en/articles/20001516-managing-usage-with-gpt-6-astra-in-work-and-codex) |
| "A new window starts when you send your first message in Work or Codex after the previous window ends." | Verified (same caveat) | same |
| Pro 100 / 200 / 500 have no five-hour limit in Work and Codex. The pricing page says "Pro plans currently have no five-hour limit." | Verified | same; [Codex pricing](https://learn.chatgpt.com/docs/pricing) (redirected from developers.openai.com/codex/pricing) |
| Per-window message counts are "not fixed message limits"; see the usage dashboard for "current limits and reset times". In the CLI, `/status` shows remaining limits. | Verified | [Codex pricing](https://learn.chatgpt.com/docs/pricing) |
| "Local messages and cloud chats share your plan's usage allowance." | Verified | same |
| The backend tracks limits as snapshots, each with a `limit_id`/`limit_name`, a `primary` window and a `secondary` window. A usage-limit error carries `resets_at` and `limit_window_minutes`. | Verified (source) | [`protocol.rs` `RateLimitSnapshot`](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/protocol/src/protocol.rs), [`error.rs` `UsageLimitReachedError`](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/protocol/src/error.rs) |
| The weekly window also anchors to first use after a reset, not to a fixed boundary. | Inference (user reports only, no staff reply) | [openai/codex#28246](https://github.com/openai/codex/issues/28246) (open) |
| Just opening the Codex **desktop app** to look at usage seemed to start a 5-hour window. | Unverified user report, no staff reply, CLI not tested | [openai/codex#27788](https://github.com/openai/codex/issues/27788) (open) |
| Community threads describe the 5-hour limit being temporarily lifted and restored during summer 2026. | Unverified (forum) | [community.openai.com](https://community.openai.com/t/clarification-on-the-temporary-removal-of-the-codex-five-hour-limit/1386911) |

**Implication:** The window is anchored to first use (this matches GLOSSARY's **Usage window**). So a warm-up at time T should make the window reset at T+5h, but only if the previous window has already expired. A warm-up sent inside a live window does nothing.

## 2. Does the cheapest model count toward the same window?

| Claim | Status | Source |
|---|---|---|
| The pricing page lists per-five-hour estimates for Plus/Business by model, all as shares of one allowance: GPT-6 Astra 5-45, GPT-6.1 Sol 15-160, GPT-6 Sol 15-150, GPT-6 Luna 350-3,000. Luna is the cheapest per message. | Verified | [Codex pricing](https://learn.chatgpt.com/docs/pricing) |
| "Model choice, context, reasoning, tool use, retrieval, and caching all affect usage." Fast/Ultrafast modes cost 2.5x / 8x. | Verified | same |
| Some models have **their own named limit bucket**. The CLI prints "You've hit your usage limit for {limit_name}. Switch to another model now" when the bucket's `limit_name` is not `codex` (the general bucket) or `gpt-reserve`. | Verified (source) | [`error.rs` L701-719](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/protocol/src/error.rs) |
| The help article title ("Managing usage with GPT-6 Astra…") suggests GPT-6 Astra is one of the models with a separate limit. | Inference | help article title |
| A message on the cheapest model (Luna) counts toward the **general `codex` bucket** and so starts the shared 5-hour window. | **Inference, not verified.** The pricing page frames all models as one allowance. No source says which `limit_id` a given model maps to. | — |

**How to verify on the host:** run the warm-up with `-m <luna-slug>`, then check `/status` (interactive) or the usage dashboard. The general Codex 5-hour reset time should read about T+5h. The model slug itself also needs confirming: list the models with the CLI's model picker on the host.

## 3. Exact non-interactive invocation

All verified from `codex exec --help` (v0.162.1), the [non-interactive mode docs](https://learn.chatgpt.com/docs/non-interactive-mode), and [`exec/src/lib.rs`](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/exec/src/lib.rs):

- **No approval prompts by default.** `exec` hard-codes `approval_policy: Some(AskForApproval::Never)` ("Default to never ask for approvals in headless mode"; lib.rs L589-591). `codex exec` has **no** `-a/--ask-for-approval` flag; only the interactive `codex` has one.
- **Sandbox defaults to read-only:** "By default, `codex exec` runs in a read-only sandbox." Pass `--sandbox read-only` explicitly anyway. Never use `--dangerously-bypass-approvals-and-sandbox` or `--approve-for-me`.
- **No git repo needed:** `--skip-git-repo-check`. Without it, exec exits 1 outside a git repo (lib.rs L1013-1018).
- **No session files:** `--ephemeral` ("Run without persisting session files to disk").
- **Prompt and stdin:** if stdin is piped and a prompt is also given, stdin is appended as a `<stdin>` block. In a local test, exec printed "Reading additional input from stdin...". Under cron, redirect `</dev/null` so it never waits on stdin.
- **Auth:** "`codex exec` reuses saved CLI authentication by default." The docs say API keys (`CODEX_API_KEY`) are "the right default for automation", but this project uses subscription login only. Do not set `CODEX_API_KEY` or `OPENAI_API_KEY` in the cron environment, and consider `-c forced_login_method="chatgpt"` ([config reference](https://learn.chatgpt.com/docs/config-file/config-reference)).
- **Cheapest settings:** `-m <model>`, `-c model_reasoning_effort="low"` (allowed values depend on the model), `-c web_search="disabled"` (the default is `"cached"`) ([config reference](https://learn.chatgpt.com/docs/config-file/config-reference)).
- **"No tool use" cannot be enforced, only discouraged.** No exec flag turns off the shell tool. The read-only sandbox and the never-approve policy keep any tool call harmless, and the prompt asks the model not to run commands. *Inference:* a trivial prompt on a low-effort model will almost never call tools.

## 4. Exit codes and output

| Situation | Behavior | Status / source |
|---|---|---|
| Output streams | Without `--json`, progress goes to **stderr** and only the final agent message goes to **stdout**. With `--json`, stdout is JSONL with events `thread.started`, `turn.started`, `turn.completed` (with a `usage` token object), `turn.failed`, `item.*`, `error`. | Verified, [non-interactive docs](https://learn.chatgpt.com/docs/non-interactive-mode) |
| Success | Exit **0** | Verified (source): `run` returns `Ok(())` when no fatal error was seen (lib.rs L1398-1406) |
| Any fatal error (turn `Failed`/`Interrupted`, or a non-retrying `Error` notification) | Exit **1**. No distinct exit codes exist per error type. | Verified (source): `error_seen` → `std::process::exit(1)` (lib.rs L1301-1404) |
| Usage limit hit | Exit **1**. The message text starts "You've hit your usage limit" (per-model variant: "…for {limit_name}. Switch to another model now") and usually ends with a retry time from `resets_at`. In JSONL, `error`/`turn.failed` carry only `{"message": "..."}`, with no machine-readable code. | Verified (source): [`error.rs`](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/protocol/src/error.rs) L701-800, [`exec_events.rs`](https://github.com/openai/codex/blob/rust-v0.162.1/codex-rs/exec/src/exec_events.rs) `ThreadErrorEvent { message }`. Not observed live. |
| Not logged in | **Tested locally** with an empty `CODEX_HOME` and `cli_auth_credentials_store="file"`. Exec does **not** fail fast. It tries the WebSocket transport 5 times (`401 Unauthorized: Missing bearer or basic authentication`), falls back to HTTPS, retries 5 more times (~10 s total), then exits **1**. Retry attempts also show up as JSONL `error` events (`"Reconnecting... n/5 (...)"`), so an `error` line alone does not mean the run failed. | Verified (local run, v0.162.1, 2026-10-12) |
| Preflight auth check | `codex login status` prints `Logged in using ChatGPT` and exits 0, or prints `Not logged in` and exits 1. | Verified (local run) |
| Expired/unrefreshable token | Maps to `CodexErrorInfo::Unauthorized` (`RefreshTokenFailed`). Exit 1 per the general rule. Exact text not observed. | Verified mapping (source), text unknown |

**Suggested classification for the wrapper** (inference): exit 0 → started. Exit 1 → grep the stderr/JSONL for `usage limit` → "limit hit, window already running". `401|Not logged in|Unauthorized` → "auth failure, needs `codex login`". Anything else → generic failure. Run `codex login status` first so auth failures show up clearly without the ~10 s of retries.

## Open questions

1. **End-to-end confirmation:** does one `codex exec` message really move the 5-hour reset time to T+5h? Run it once on the host after a window has expired and read `/status` or the usage dashboard.
2. **Which bucket the cheapest model uses:** does Luna (or whatever the cheapest slug is) count against the general `codex` limit, or a separate named `limit_id`? If it uses a separate bucket, the warm-up must use a model in the general bucket.
3. **Does a failed request start a window?** For example, a usage-limit rejection or a network error mid-turn. Unknown.
4. **Does the CLI start a window without a message?** #27788 suggests opening the desktop app may start one. If CLI startup or `codex login status` did the same, timing could drift. Untested.
5. **Is the 5-hour limit active on the user's plan right now?** It does not exist on Pro, and community posts describe temporary removals.
6. Live usage-limit stderr/JSONL output has not been observed. The wording above comes from source and may change between releases.
