# Ex7 — Handoff bridge

## Your answer


I ran this against the **real** Rasa CALM server (`sessions/sess_769562b1f1cf`)
and it completed the full round-trip in 2 rounds: round 1 proposed a party of
12, which CALM rejected as `party_too_large`, so the bridge built a reverse
task; round 2 proposed a party of 6, which CALM confirmed. (`make ex7-real`
isn't a target — I ran `python -m starter.handoff_bridge.run --real`, which
keeps the loop scripted and makes only Rasa live.)

Fail-closed IPC: after a rejection the bridge archives the forward handoff out
of `ipc/` before looping, so at most one handoff file is ever live. The Ex7
integrity check then audits the trace for real rounds, state transitions, and
tool calls — it passed here, which is what catches a bridge that claims
`completed` without doing real work.

## Citations

- `sessions/sess_769562b1f1cf/logs/trace.jsonl:6-7,13-14` (real loop↔structured round-trip + `party_too_large`)
- `starter/handoff_bridge/bridge.py:147-152` (fail-closed handoff archive)
- `starter/handoff_bridge/integrity.py` (trace audit)
