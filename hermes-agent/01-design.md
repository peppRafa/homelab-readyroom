# Design

## Goals

1. Run an agent on a local LLM — no cloud inference dependency
2. Use the HPE server as the "brain" — dedicated hardware, not a shared workstation
3. Continue a conversation seamlessly across devices (phone, laptop) when away from home
4. Support two fully isolated agents eventually (two household users), each with
   separate model instance, memory, and tool access
5. Use MCP for tool integration
6. Document the build for GitHub and LinkedIn

## Key decisions

### Framework: Hermes Agent, not a custom Open WebUI + mcpo stack

Initial plan was Ollama + Open WebUI (for chat UI, multi-device history) + `mcpo`
(bridging MCP servers into Open WebUI's tool format). Partway through design,
identified that **Hermes Agent** (Nous Research, MIT licensed, self-hosted) already
provides all of this natively:

- Persistent, server-side conversation memory (device continuity is free)
- Native MCP client support — no bridge/proxy layer needed
- Built-in **Profiles** feature for isolated multi-agent setups
- CLI, web dashboard, and messaging-gateway entry points

This removed an entire layer (Open WebUI + mcpo) from the architecture.

### Model: Qwen 2.5 7B (Q4_K_M), not Hermes 4 14B

Evaluated Hermes 4 (14B and 70B, Nous Research's own model line) against Qwen 2.5 7B.

- **Hermes 4 14B** — better fidelity, native alignment with Hermes Agent's tool-calling
  format, but ~6-9 tok/s estimated on this CPU
- **Hermes 4 70B** — too slow for interactive tool-calling loops on CPU-only hardware
  (~1-2 tok/s estimated)
- **Qwen 2.5 7B** — faster, smaller footprint, mature/well-tested, but weaker reasoning
  depth and no native alignment with Hermes Agent's prompt format

**Chosen: Qwen 2.5 7B**, on the basis that model swapping is a config change, not a
re-architecture (`hermes model` / editing `config.yaml`). Start fast and cheap to
validate the full pipeline (Ollama → Hermes Agent → MCP), upgrade the model later
if fidelity on real project work proves insufficient.

Explicitly **not pursuing DS4/DeepSeek V4 Flash** for this deployment — DS4
(antirez's inference engine) targets GPU/unified-memory hardware only, and while
mainline llama.cpp gained CPU-capable DeepSeek V4 support during this project window,
no CPU benchmark data exists yet and RAM footprint at usable quants is tight. Flagged
as a future experiment, not part of this build.

### Remote access: existing WireGuard tunnel, not Tailscale

Already had a working, hardened WireGuard tunnel (split tunnel, AdGuard DNS, DDNS).
Tailscale's core value — NAT traversal without port forwarding — solves a problem
this setup doesn't have. Added complexity with no capability gain.

### Multi-agent isolation: dedicated Linux user per agent

Rather than running Hermes Agent's Profiles feature under one shared account,
each agent gets its **own system user** (`hermes-captain`, and a second user later
for the second agent). Reasons:

- Genuine security boundary — Hermes Agent's tool set includes shell execution;
  a scoped, non-sudo service account limits blast radius
- `$HERMES_HOME` (`~/.hermes`) is per-user by default, so this is the natural
  isolation unit
- Matches Hermes Agent's own Profiles pattern, just at the OS level instead of
  within one account

### OS: Ubuntu 24.04 LTS bare metal, no hypervisor

CPU-only inference is memory-bandwidth-bound (see `03-benchmark.md`); a hypervisor
layer taxes exactly the resource that's already the constraint. Considered
Ubuntu 26.04 LTS (newer, longer support window) but stayed on 24.04 for maturity
and to avoid the `sudo-rs`/`uutils` coreutils replacement affecting existing
automation scripts.

## Deferred to a later phase

- Second agent profile (wife's agent) — same mechanism, once this one is validated
- Web dashboard / gateway exposure over WireGuard — CLI-only for now
- Messaging platform gateway (Telegram/Signal/etc.) — deliberately skipped to avoid
  routing conversations through third-party platforms
