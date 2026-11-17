# AI-RAN Edge A2A Agent

An emulated on-device agent, and the orchestration logic that decides whether
a reported network issue can be resolved from device-side signals alone or
needs to be escalated to a NOC agent -- scoped from feedback given by AI-RAN
Alliance WG3 members on the [Closed-Loop RAN Care](https://github.com/athpra/closed-loop-ran-care)
concept.

This is a separate repo from the two prototypes it connects to, because its
scope is different: those two are call-handling and network-investigation
applications; this one is about *where inference runs* (edge vs. centralized)
and *how two agents agree to talk to each other* (A2A-T). It depends on
neither of their codebases.

## How this relates to the other repos

- **[voice-ai-agent](https://github.com/athpra/voice-ai)** -- handles the real
  phone call. In the target design, it queries the device agent here mid-call
  instead of (or alongside) its own static `home_cell_id` lookup. That wiring
  is a future change to `voice-ai-agent` itself, not part of this repo.
- **[mobile-noc-agent-demo](https://github.com/athpra/mobile-noc-agent-demo)**
  -- the network investigation agent. `orchestrator/noc_client.py` is a stub
  for calling it, since that repo doesn't expose an API yet.
- **[closed-loop-ran-care](https://github.com/athpra/closed-loop-ran-care)**
  -- the concept note and deck this work was scoped from (see Section 08,
  "Feedback from WG3, and what's next").

## What's in this repo

- `device_agent/` -- an emulated on-device agent: synthetic UE-side radio
  signals (serving cell, RSRP/RSRQ, recent handover/retry/drop events),
  served over a simplified, illustrative approximation of TM Forum's
  **A2A-T** (`IG1453`/`IG1453A`) message shape. See the caveat in
  `docs/architecture.md` -- this has not been verified against the published
  spec text.
- `orchestrator/` -- queries the device agent, decides whether the signals
  are enough to resolve the issue outright or need NOC escalation, and (for
  escalation) calls the `mobile-noc-agent-demo` stub.
- `demo/run_demo.py` -- a standalone CLI that drives the whole thing against
  a scripted "caller complaint," with no dependency on a live phone call.
- `benchmarks/stt_tts_latency.py` -- times each leg of the STT/TTS providers
  `voice-ai-agent` already uses (Deepgram STT, Cartesia STT/TTS), to put real
  numbers behind the edge-inferencing latency argument.
- `docs/architecture.md` -- the orchestration design and the A2A-T caveat.
- `docs/device-log-model.md` -- the synthetic device-signal schema, modeled
  on 3GPP's UE-side mechanisms (Minimization of Drive Tests, UE Information
  Report), since no TM Forum spec covers device-log content.

## Running the demo

```bash
pip install -r requirements.txt
python demo/run_demo.py --scenario self-resolve
python demo/run_demo.py --scenario escalate
```

Both scenarios are synthetic and run entirely in-process -- no API keys
needed.

## Running the STT/TTS benchmark

Needs real provider credentials, since it makes live calls:

```bash
export DEEPGRAM_API_KEY=...
export CARTESIA_API_KEY=...
python benchmarks/stt_tts_latency.py
```

## Status

Everything here is new, unscoped-until-now work coming out of WG3 feedback.
None of it is "proven" in the sense `closed-loop-ran-care`'s proven/proposed
table uses that word -- the device agent is synthetic, the A2A-T message
shape is illustrative, and `noc_client.py` is a mock.
