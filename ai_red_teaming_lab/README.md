# AI Red Teaming Lab

A local environment for testing large language model safety and running red team probes against a model you control end to end.

## What this is

This repository documents setting up a fully local AI red teaming stack: a target model running through Ollama, and Garak, an automated LLM vulnerability scanner, testing that model for jailbreaks and unsafe behavior. Everything runs on your own machine, so there is no reliance on a hosted API, no logging by a third party, and no rate limits.

## Stack

| Tool | Role | Notes |
|---|---|---|
| Ollama | Runs the target model locally | Tested on Windows 11 |
| Qwen3 | The target model being probed | 7B parameter version used here |
| Garak | Automated LLM vulnerability scanner | Run from WSL/Ubuntu against the Windows hosted Ollama instance |

## Hardware this was tested on

CPU only laptop, no dedicated GPU (Intel Iris Xe integrated graphics), 16GB RAM. If your hardware looks similar, expect generations to take much longer than on a GPU equipped machine. Budget for scans that run for hours rather than minutes.

## Repo structure

```
ai_red_teaming_lab/
  README.md
  docs/
    01_install_ollama_and_qwen3.md
    02_garak_setup_journey.md
  LICENSE.md
  .gitignore
```

## Quick start

1. Follow `docs/01_install_ollama_and_qwen3.md` to install Ollama and pull the Qwen3 model.
2. Follow `docs/02_garak_setup_journey.md` to set up Garak inside WSL and point it at your local Ollama instance. This document also covers the exact bugs hit along the way, including a hardcoded timeout that is too short for slow hardware and a config setting that silently fails to apply, plus how each one was resolved.

## Roadmap

Planned additions to this repository: the actual scan reports with their interpretation, and a set of hardening recommendations based on what the scans found.

## License

MIT. See `LICENSE.md`.

---

Written as part of the AI red teaming section on cyberlav.io.
