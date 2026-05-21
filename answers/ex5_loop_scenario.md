# Ex5 — Edinburgh research loop scenario

## Your answer

Offline (`make ex5`) completes: flyer written, "dataflow OK: verified 4
facts." I confirmed the guard directly — mutating `£540`→`£9999` in the flyer
yields `dataflow FAIL: 1 unverified fact(s): ['£9999']`.

Tested against real Nebius. In `sessions/sess_d33f553a7bfe` the executor
spiralled: four `venue_search` calls (`trace.jsonl:3-6`) with a hallucinated party size of 50, zero results each time, then a handoff — no flyer. `sess_63d9cfe78440` spiralled differently (party 6 but `near='Edinburgh'`, which doesn't substring-match the fixture's district names). The offline `FakeLLMClient` hides this because the tool args are scripted; the real model, given an under-specified payload, fills the gap with wrong guesses.

## Citations

- `sessions/sess_d33f553a7bfe/logs/trace.jsonl:3-7` (real 4× venue_search spiral, party=50, no flyer)
- `sessions/sess_63d9cfe78440/logs/trace.jsonl` (second spiral: wrong `near` values)
- `starter/edinburgh_research/tools.py`, `integrity.py:118-164` (`verify_dataflow`)
