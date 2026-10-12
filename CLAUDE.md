# cc-warmup

A small bash utility, run from cron on an always-on Linux host, that sends scheduled warm-ups to Claude Code and Codex so their 5-hour usage windows start at chosen times. See `GLOSSARY.md` for terms.

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `haoyan-dev/cc-warmup`, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `GLOSSARY.md` plus `docs/adr/`. See `docs/agents/domain.md`.
