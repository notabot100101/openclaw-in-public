# openclaw-in-public

Sanitized, build-in-public engineering notes and clean-room reference patterns
from building on top of [OpenClaw](https://github.com/openclaw/openclaw).

No metrics theater — just real reliability and tooling fixes we shipped, written
up so they're useful to anyone building multi-agent systems.

## Writeups
- [**A slow agent turn is not a failed turn**](writeups/slow-turn-is-not-a-failed-turn.md) — treat a timed-out agent handoff as *deferred*, not failed: self-assign the session id and read the completed answer on the next run.
- [**DeepSeek tool calls silently vanishing when routed via OpenRouter**](writeups/deepseek-tool-calls-silently-vanishing.md) — a subtle routing bug where a native tool-call format was silently dropped; the runtime already had the fix, disabled by an auto-detected default. One scoped config line.
- [**A fallback that can also fail is not a safety net**](writeups/model-fallback-resilience-chain.md) — give every agent a local last-resort model so it never fully stalls during a cloud outage, and watch for per-agent overrides that silently drop the default floor.

## Licensing
- **Code** (snippets in the writeups): MIT — see [`LICENSE-CODE`](LICENSE-CODE).
- **Prose** (the writeups): CC-BY-4.0 — see [`LICENSE-DOCS`](LICENSE-DOCS).
- **Attribution / notices**: see [`ATTRIBUTION.md`](ATTRIBUTION.md). Built around OpenClaw (MIT); this repo is independent and not endorsed by OpenClaw.

## Clean-room rule

Everything in this repo is deliberately sanitized, clean-room material. It must
never contain:

- **Private repo content** — nothing from the private ops repo or its internals,
  and nothing from any deploy/automation/production repositories.
- **Credentials** — no tokens, API keys, passwords, or any other secrets.
- **Personal data** — no Gmail addresses, private logs, or personally identifying
  information.

If a writeup needs to reference an internal detail, it is generalized and
rewritten so the public version stands on its own.
