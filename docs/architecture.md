# Architecture

## End-to-end flow

![End-to-end architecture: a caller's complaint flows from voice-ai-agent into this repo's orchestrator, which queries the emulated device agent over A2A-T, then either resolves the issue directly or escalates to mobile-noc-agent-demo for human approval, before a resolution is spoken back to the caller.](assets/architecture.svg)

Solid lines are built and exercised today (this repo's demo, or the existing deployed apps). Dashed lines are the integration points that aren't wired yet -- into `voice-ai-agent`'s call flow, and into `mobile-noc-agent-demo` -- plus `noc_client.py`'s mock. The teal lines are the A2A-T exchange between the orchestrator and the emulated device agent -- the one piece of new protocol work this repo actually demonstrates.

`demo/run_demo.py` drives the middle of this end to end with no network calls and no dependency on a live phone call: it stands in for the "caller complaint" that would otherwise come from `voice-ai-agent`'s call transcript, and stops at the decision (self-resolve or escalate) rather than actually speaking back to a caller or paging a NOC engineer.

## The two paths

1. **Self-resolve.** The device agent's signals directly explain the
   complaint (e.g. a known handover/PCI pattern on the caller's serving
   cell), so `decision.py` returns a resolution without involving a NOC
   agent at all.
2. **Escalate.** The device signals are inconclusive or point to something
   outside what device-side telemetry alone can diagnose (e.g. a
   backhaul/config-level fault), so the orchestrator hands off to
   `noc_client.py`.

`decision.py`'s rule today is a simple, explicit pattern match -- not an LLM
call -- to keep this repo's first version self-contained and auditable. The
real reasoning step (likely reusing `voice-ai-agent`'s LLM, or the NOC
agent's own LangGraph reasoning) is a deliberate later swap-in, not built
here yet.

## Why A2A-T, and the caveat on fidelity

TM Forum released **A2A-T** (`IG1453`/`IG1453A`) at Accelerate Week in
February 2026 -- a telecom-specific extension of the Linux Foundation's A2A
protocol, built for network-layer concerns (reliability, security, latency)
rather than plain A2A's enterprise-software framing. Adopting it here means
citing a live TM Forum initiative for agent-to-agent communication instead
of inventing a bespoke one.

**Caveat:** `device_agent/a2a_server.py`'s message envelope is a simplified,
illustrative approximation of what A2A-T message/task exchanges are
*described* as doing (agent cards, tasks, artifacts), built from public
descriptions of the protocol -- not from the normative `IG1453` document
text, which wasn't available to verify against while building this. Treat
the wire format here as a placeholder shape to replace once the actual spec
is reviewed, not as a conformant implementation.

## Why device-side signals are modeled on 3GPP, not TM Forum

No TM Forum specification covers UE-side log or radio-diagnostic content --
`TMF908` (IoT Agent and Device Component Suite) is an IoT device-management
spec and the wrong shape. `docs/device-log-model.md` instead borrows field
concepts from 3GPP's own UE-side mechanisms (Minimization of Drive Tests,
UE Information Report, RRC measurement reports).

## What's intentionally out of scope here

- Wiring `voice-ai-agent`'s real call flow to call this orchestrator mid-call
  -- a future change to that repo, not this one.
- Giving `mobile-noc-agent-demo` a real API for `noc_client.py` to call --
  a future change to that repo, not this one.
- Verifying the A2A-T message shape against the actual `IG1453` spec text.
- Any real on-device deployment -- `device_agent` is a synthetic emulation,
  not something that runs on a handset.
