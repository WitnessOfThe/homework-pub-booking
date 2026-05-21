# Ex9 — Reflection
## Q1 — Planner handoff decision

### Your answer

In my Ex7 round-trip (`sessions/sess_769562b1f1cf`), the handoff to the
structured half is visible at `logs/trace.jsonl:5-6`. Line 5 is an
`executor.tool_called` event for `handoff_to_structured`; line 6 is the
bridge's `session.state_changed` with `{"from": "loop", "to": "structured",
"round": 1}`. The planner had kept research local — the round-1 plan's subgoal
carries `assigned_half: "loop"`, not `"structured"` — so the *planner* never
routed work to the structured half. The decision to cross the boundary was the
loop half emitting `next_action = handoff_to_structured` once a candidate venue
was identified, and the `HandoffBridge` acting on that signal.

So the precise signal is the loop's terminal action after venue identification:
the agent reached the policy-confirmation boundary (a chosen venue that now
needs rule-checking) and chose to delegate rather than decide policy itself.
The bridge then wrote `ipc/handoff_to_structured.json` and logged the
transition; real Rasa CALM rejected round 1 with
`rejection_reason: "...reason: party_too_large"` (`trace.jsonl:7`), which the
bridge turned into a reverse handoff for round 2.

Worth contrasting with a *genuinely LLM-driven* handoff I saw in Ex5
(`sessions/sess_a54bf2c673ed/logs/trace.jsonl:7`): there the real executor chose
`handoff_to_structured` with the reason "Missing required parameters for cost
calculation" — a model decision, not a scripted one. The Ex7 runner scripts the
loop (`FakeLLMClient`), so its handoff reason is fixed; the architectural point
is the same — `assigned_half` and `next_action` are the two routing signals, and
only `next_action` actually moved control here.

### Citation

- `sessions/sess_769562b1f1cf/logs/trace.jsonl:5-7` (handoff event + loop→structured + rejection_reason)
- `sessions/sess_a54bf2c673ed/logs/trace.jsonl:7` (a real model-driven handoff for contrast)

---

## Q2 — Dataflow integrity catch

### Your answer

I'll take the "plausible scenario" option honestly: across three real Ex5 runs
the loop half never produced a flyer (it spiralled or handed off — see Q3), so
`verify_dataflow` never ran on real output. But one run got far enough to make
the failure it guards against concrete. In `sessions/sess_a54bf2c673ed` the real
executor fetched genuine data — `venue_search` returned `haymarket_tap`
(`trace.jsonl:4`) and `get_weather` returned `"condition": "cloudy"`
(`trace.jsonl:6`) — before bailing on the cost step.

Construct the test from that state: let the model proceed to `generate_flyer`
but, instead of the £556 that `calculate_cost` logs, have it write a *plausible*
"Total: £612" (LLMs round and drift). A human reviewer skims £612 next to a
party of six and a Haymarket pub and accepts it — it looks right.
`verify_dataflow` does not reason about plausibility: it strips the HTML, regexes
`£612`, normalises it, and scans every `_TOOL_CALL_LOG` record's `output` and
`arguments` for a matching scalar. £556 is there; £612 is not in any record, so
it returns `ok=False, unverified_facts=['£612']` and names the offending value.

I verified the mechanism directly: with the log populated by a real tool run,
mutating the flyer's `£540`→`£9999` produced exactly
`dataflow FAIL: 1 unverified fact(s): ['£9999']`, while the unmodified flyer
passed with 4 verified facts. The catch works because provenance, not
reasonableness, is the test — a number is legitimate only if some tool produced
it.

### Citation

- `sessions/sess_a54bf2c673ed/logs/trace.jsonl:4,6` (real venue + weather facts that would seed a flyer)
- `starter/edinburgh_research/integrity.py:99-164` (`fact_appears_in_log` / `verify_dataflow`)

---

## Q3 — First production failure + the primitive that surfaces it

### Your answer

**Failure mode (one):** the loop half *spirals* — it repeats one tool with
drifting arguments, makes no progress, and terminates without producing the
deliverable. I hit this on the first real run, not as a hypothetical. In
`sessions/sess_d33f553a7bfe` the executor called `venue_search` four times
(`trace.jsonl:3-6`) with a hallucinated party size of 50 and escalating
budgets, got zero results every time, then handed off (`trace.jsonl:7`) — no
flyer written. A second run (`sess_63d9cfe78440`) spiralled differently:
party 6 but `near='Edinburgh'/'Old Town'`, which don't substring-match the
fixture's district names, so again zero results. Shipping this to a real
business, the first incident is "the agent ran, charged tokens, and returned
nothing."

**Primitive that surfaces it (one):** **manifest discipline.** Each operation
writes a ticket whose manifest records the artifacts it produced. The
`executor.run_subgoal/sg_1` ticket can report `state: success` while its
manifest contains **no `flyer.html`** — the run-time check in
`starter/edinburgh_research/run.py` keys off exactly this (`workspace/flyer.html`
absent → "Ex5 failed", exit 1). In production that empty-manifest signal is what
turns a silent non-delivery into an alert: you don't trust the agent's "done,"
you trust the manifest of what it actually wrote. A green ticket with an empty
artifact manifest is the tripwire.

(The complementary defence is a tool-call cap after N identical calls, per
`docs/real-mode-failures.md` — but the primitive that *surfaces* the failure is
manifest discipline.)

### Citation

- `sessions/sess_d33f553a7bfe/logs/trace.jsonl:3-7` (4× venue_search spiral → handoff, no flyer)
- `sessions/sess_63d9cfe78440/logs/trace.jsonl` (a second, differently-shaped spiral)
- `starter/edinburgh_research/run.py:259-278` (the missing-flyer / empty-artifact check)
