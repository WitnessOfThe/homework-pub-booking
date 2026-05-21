# Real-mode runs — what actually happened

Live runs against Nebius (Qwen3-Next-80B planner, Qwen3-32B executor,
Llama-3.3-70B persona/manager) and a real Rasa Pro 3.16.4 CALM server. Keys
reused from `../soverign_agent_lab/.env`. Sessions live under
`~/.local/share/sovereign-agent/`. **These `sess_…` ids are the evidence Ex9
must cite** — copy the relevant ones under `./sessions/` and `git add -f` them.

> Big picture: offline/mock (what `make check-submit` grades) passes cleanly,
> but **real mode exposes failures the scripts hide**. That gap *is* the
> curriculum.

---

## Ex5 — the loop half spirals (3/3 runs failed to write a flyer)

| Session | What the real executor did | Why |
|---|---|---|
| `ex5-edinburgh-research/sess_d33f553a7bfe` | `venue_search` ×4 with **party_size 50/50/40/30**, then `handoff_to_structured` | hallucinated party size; 0 results every time → spiral |
| `ex5-edinburgh-research/sess_63d9cfe78440` | `venue_search` ×3 with party 6 but `near='Edinburgh'/'Old Town'/'City Centre'`, budgets 200/300/500 | `near` is a substring match on the venue's `area` (district); "Edinburgh" matches no district; low budgets exclude venues → 0 results → spiral |
| `ex5-wellspecified/sess_a54bf2c673ed` | `venue_search`→haymarket_tap ✓, `get_weather`→cloudy/12 ✓ (**real data**), then `handoff_to_structured` on the cost step claiming *"Missing required parameters for cost calculation"* | executor had every param (`venue_id`, `party_size`, `duration_hours` default) but wrongly self-assessed it couldn't proceed |

**Root cause of runs 1–2:** the runner invokes the loop half with a terse
payload `{"task": "research Edinburgh venue and write flyer"}`
([run.py:250](../../starter/edinburgh_research/run.py)); the real constraints
(party 6, area Haymarket, the 5-step tool sequence) live in `session.task` and
never reach the model. Offline the scripted `FakeLLMClient` doesn't care; the
real model fills the gap with plausible-but-wrong guesses.

**Lesson:** under-specified prompts + a tool whose filter you don't know →
0-result spiral. The defensive pattern (tool-call cap after N calls) in
`docs/real-mode-failures.md` exists for exactly this. The dataflow-integrity
check never even ran (no flyer) — a *different* failure class than fabrication.

---

## Ex6 — real Rasa CALM (all three rules verified live)

Rasa Pro 3.16.4, model trained from `rasa_project/`, served on :5005 with the
action server on :5055.

| Input | Real CALM result |
|---|---|
| party 6, deposit £200 (`ex6-rasa-half/sess_2baa3197911b`) | **confirmed**, `utter_booking_confirmed`, ref **BK-7D401E9E** |
| party 6, deposit £500 (`ex6-deposit-probe/sess_230c638ad40a`) | **rejected**, `deposit_too_high` → `escalate` |
| party 12 (in the Ex7 round-trip below) | **rejected**, `party_too_large` → `escalate` |

The full path worked: validator normalised `"25th April 2026"`→`2026-04-25`,
`"7:30pm"`→`19:30`, `"Haymarket Tap"`→`haymarket_tap`, `"£200"`→`200`; the
LLM command generator turned `/confirm_booking` into `StartFlow`;
`action_validate_booking` read `metadata.booking`, applied the rules, set slots;
the flow uttered the right response. Identical verdicts to the stdlib mock —
which is the point of the mock.

---

## Ex7 — real round-trip bridge (`ex7-handoff-bridge/sess_769562b1f1cf`)

Scripted loop + **real Rasa CALM**. Outcome: `completed`, **2 rounds**. Trace:

```
bridge.round_start                       round=1
session.state_changed  loop->structured  round=1
session.state_changed  structured->loop  round=1  reason="...reason: party_too_large"   ← real CALM reject
bridge.round_start                       round=2
session.state_changed  loop->structured  round=2
session.state_changed  structured->complete round=2                                      ← real CALM confirm
```

Ex7 integrity audit: **OK** (2 round_starts, 4 state_changes, 4 tool_calls).
This `sess_769562b1f1cf` is ideal **Ex9-Q1** material — a real
`session.state_changed loop→structured` with the actual rejection reason from
the live dialog engine.

> Note: `make ex7-real` is **not** a Makefile target (README drift); run
> `uv run python -m starter.handoff_bridge.run --real`. Also, this runner always
> uses the scripted `FakeLLMClient` for the loop even with `--real` — only the
> Rasa side becomes real ([run.py:142](../../starter/handoff_bridge/run.py)).

---

## Ex8 — text mode real, voice not runnable here

- **Text** (`homework/ex8/sess_fa65441a842e`): real Llama-3.3 "Alasdair" held a
  3-turn booking chat, in character ("Aye, we can do that... half seven"),
  accepted party 6 + £150 deposit per the rules. Trace has **3 `voice.utterance_in`
  + 3 `voice.utterance_out`** → clears the grader bar.
- **Missing-key degradation** (`homework/ex8/sess_0e0702086906`): with
  `SPEECHMATICS_KEY` unset, `--voice` warned and fell back to text cleanly — the
  graded path works.
- **Voice with key but no PortAudio** (`sess_342933a795e1`): **crashed** with
  `OSError: PortAudio library not found`. The degradation code only catches
  `ImportError` ([voice_loop.py:99-117](../../starter/voice_pipeline/voice_loop.py)),
  not the `OSError` that `import sounddevice` raises when the system PortAudio lib
  is absent (WSL2). A real gap: keys-present + audio-stack-missing isn't handled.
  Voice needs `libportaudio2` + a mic + a `RIME_API_KEY` (none present here).

---

## Spend / environment notes

- `.env` here reuses the user's `NEBIUS_KEY`, `RASA_PRO_LICENSE`, and
  `SPEECHMATICS_KEY` (← lab's `SPEECHMATICS_API_KEY`); no `RIME_API_KEY`.
- `make verify` smoke step fails on a hard-coded `google/gemma-2-2b-it` (404 —
  not on this Nebius endpoint); the key is fine — `Qwen3-32B`,
  `Qwen3-Next-80B-A3B-Thinking`, and `Llama-3.3-70B-Instruct` all respond.
  Workaround: `NEBIUS_SMOKE_MODEL=Qwen/Qwen3-32B make verify`.
- Total spend was a few pennies of Nebius tokens.
