---
type: fact
tags: [hermes, 9router, models]
created: 2026-07-22
agent: main
---

Hermes connects to the local 9router OpenAI-compatible endpoint through the named `custom:9router` provider in `~/.hermes/config.yaml`. A bare `model.provider: custom` entry only exposed the current model in Hermes' picker; `custom_providers` with live discovery exposes the router catalog. The default verified model is `cx/gpt-5.6-sol`. Keep the credential in `~/.hermes/.env` and do not copy it into memory notes.
