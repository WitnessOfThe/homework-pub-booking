# Ex6 — Rasa structured half

## Your answer

Ex6 runs Rasa CALM as the structured half that confirms/rejects bookings against policy. The committed evidence is the Ex7 round-trip (`sessions/sess_769562b1f1cf`),
which drives real CALM both ways: round 1 (party 12) hit
`validation_error = party_too_large` → `utter_booking_rejected`, logged as
`structured→loop` with `reason: party_too_large` (`trace.jsonl:7`); round 2
(party 6) was confirmed → `structured→complete` (`trace.jsonl:14`). A standalone
Ex6 run additionally returned reference `BK-7D401E9E` for party 6 / £200, and a
£500 deposit returned `deposit_too_high` — both verified live, though those
single HTTP-call runs don't persist a trace.

The validator's normalisation showed up live too: `"25th April 2026"`→
`2026-04-25`, `"7:30pm"`→`19:30`, `"Haymarket Tap"`→`haymarket_tap`,
`"£200"`→`200`. The stdlib mock replicates the same thresholds, so offline tests
give identical verdicts; failures map to `escalate` (validation) or
`SA_EXT_SERVICE_UNAVAILABLE`/`SA_EXT_TIMEOUT` (network), never a crash.

## Citations

- `sessions/sess_769562b1f1cf/logs/trace.jsonl:7,14` (real CALM reject + confirm)
- `rasa_project/actions/actions.py:29-30,119-123` (party ≤ 8 / deposit ≤ 300 rules)
- `starter/rasa_half/validator.py`, `structured_half.py` (normalise + `run`)
