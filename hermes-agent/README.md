# Hermes Agent — Local LLM Deployment on Enterprise Hardware

A self-hosted, fully local AI agent running on repurposed enterprise server hardware —
no cloud inference, no third-party data exposure, reachable from any device over a
private VPN tunnel.

## What this is

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) running
against a locally-hosted Qwen 2.5 7B model via Ollama, deployed on an HPE ProLiant
ML110 Gen10 that would otherwise be heading to resale. The goal: a private, always-on
assistant for daily technical work — coding help, homelab operations, research — with
zero reliance on external LLM providers.

## Hardware

| Component | Spec                                                                  |
|-----------|-----------------------------------------------------------------------|
| Server    | HPE ProLiant ML110 Gen10 ("M2")                                       |
| CPU       | Intel Xeon Gold 5222 — 4 cores / 8 threads, 3.8GHz base, **105W TDP** |
| RAM       | 96GB DDR4, 6-channel                                                  |
| Storage   | RAID 1 (2x SAS, mirrored), remainder left unspun                      |
| GPU       | None — CPU-only inference                                             |

## Software stack

| Layer            | Choice                                                   |
|------------------|----------------------------------------------------------|
| OS               | Ubuntu 24.04 LTS, bare metal (no hypervisor)             |
| Inference engine | Ollama (llama.cpp backend)                               |
| Model            | Qwen 2.5 7B, Q4_K_M quant                                |
| Agent framework  | Hermes Agent v0.18.2 (Nous Research)                     |
| Isolation        | Dedicated Linux system user per agent (`hermes-captain`) |
| Remote access    | Existing WireGuard tunnel (no Tailscale)                 |
| Tool integration | MCP, configured natively in Hermes Agent                 |

## Status

🚧 **In progress.** Core agent is installed and responding; one open issue (model
context window — see `02-build.md`) is still being resolved before this is called
done. A second, fully isolated agent profile for a second household user is planned
as a follow-on deployment once this one is validated.

## Docs in this folder

- [`01-design.md`](./01-design.md) — goals, architecture decisions, and alternatives considered
- [`02-build.md`](./02-build.md) — step-by-step deployment log
- [`03-benchmark.md`](./03-benchmark.md) — performance validation and hardware investigation
- `04-lessons.md` — pending, written after real-world daily use

## Why this exists

Written up as a portfolio piece documenting hands-on infrastructure and LLM deployment
work — part of an ongoing homelab build targeting NOC/infrastructure engineering roles.
