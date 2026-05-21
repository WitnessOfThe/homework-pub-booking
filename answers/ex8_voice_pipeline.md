# Ex8 — Voice pipeline

## Your answer

Text mode (`run_text_mode`) reads stdin and
the `ManagerPersona` (real Llama-3.3-70B "Alasdair", `temperature=0`) replies;
voice mode (`run_voice_mode`) adds Speechmatics STT and Rime TTS. Both emit
`voice.utterance_in` (actor user) and `voice.utterance_out` (actor manager) with
payload `{text, turn, mode}`, so grading is mode-agnostic.

I ran text mode for real (`sessions/sess_fa65441a842e`): a 3-turn booking chat
where the persona stayed in character ("Aye, we can do that. I'll pencil you in
for Friday at half seven. What's the contact number?") and accepted a party of 6
with a £150 deposit, per the rules. The trace carries exactly 3
`voice.utterance_in` + 3 `voice.utterance_out` — clearing the ≥3-turn bar.

## Citations

- `sessions/sess_fa65441a842e/logs/trace.jsonl` (3-turn real conversation, 3× in / 3× out)
- `starter/voice_pipeline/voice_loop.py:99-117` (degradation branch; the uncaught `OSError`)
- `starter/voice_pipeline/manager_persona.py:22-41` (Alasdair system prompt + rules)
