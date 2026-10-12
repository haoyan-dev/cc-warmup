# cc-warmup

Starts AI coding services' usage windows at times the user chooses, so fresh quota lands when it's most useful.

## Language

**Provider**:
An AI coding service with its own usage limits; in scope are Claude Code and Codex.
_Avoid_: Service, agent, vendor

**Usage window**:
A provider's rolling 5-hour quota period, which begins with the first message sent after the previous window has expired.
_Avoid_: Session, 5-hour limit, block

**Warm-up**:
One minimal, real message sent through a provider's official CLI to deliberately start a usage window at a chosen time.
_Avoid_: Ping, keep-alive, primer

**Schedule**:
The set of times at which warm-ups fire.
_Avoid_: Timer, cron (cron is one mechanism, not the concept)
