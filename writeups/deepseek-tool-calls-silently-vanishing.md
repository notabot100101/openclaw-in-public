# DeepSeek tool calls silently vanishing when routed via OpenRouter

*Build-in-public notes on a subtle routing bug: the fix was one config line — but
finding it meant reading the runtime's compiled source.*

## The symptom

Two weird things started happening, rarely (~1 in 180 tool-using turns), always on
the same setup — **DeepSeek's V4 Flash model, reached through OpenRouter** as a
fallback:

1. **A tool would silently not run.** The turn ended normally (`stopReason: stop`),
   the message had a text part but **no parsed tool call**. Whatever the model
   meant to *do* — search memory, run a check — just didn't happen. Silent no-op.
2. **The raw tool-call text became the "answer."** Instead of executing, the
   model's tool-call syntax showed up verbatim as the assistant's reply — the kind
   of thing a downstream step could cheerfully send to a user.

Both failure modes were **100% correlated with DeepSeek-via-OpenRouter**, and zero
other models ever did it.

## The dig

The model wasn't malformed. DeepSeek emits its tool calls in **its own native
delimiter format** — framed with fullwidth-bar `｜DSML｜` tokens, not the
mainstream `<tool_call>…` shape most parsers expect. Our gateway's tool-call
parser didn't recognize that framing, so it saw "just text," and the call fell on
the floor.

The obvious fix would be "teach the parser DeepSeek's format." So we went to add
it — and discovered the runtime **already had** a complete handler for exactly
this: both a filter that strips the raw `｜DSML｜` delimiters out of the visible
text, and a **recoverer** that parses that framing back into a real, executable
tool call.

So why wasn't it running?

## The root cause

Both the filter and the recoverer were gated behind a single check:

```
shouldHandle(compat)  →  compat.thinkingFormat === "deepseek"
```

And `thinkingFormat` was **auto-derived from the endpoint**. Hitting DeepSeek's own
API directly, it resolved to `"deepseek"` and everything worked. But routed through
**OpenRouter**, the exact same model resolved to `"openrouter"` — so the gate was
false, and the recoverer + filter were **both switched off**. The capability was
right there, disabled by a wrong auto-detected default for our routing path.

That's the whole bug: not a missing feature, a **mis-defaulted** one.

## The fix

The runtime let a per-model config value **override** the auto-detected format. So
the fix was one scoped line on that model's entry:

```jsonc
// scoped to THIS model id only — never provider-wide
{
  "id": "deepseek/deepseek-v4-flash",
  "compat": { "thinkingFormat": "deepseek" }   // force the right handler on
}
```

Zero code. Survives runtime updates (it's config, not a patch). Applied it,
restarted, and verified live: fresh tool-using probes now produced real, parsed
tool calls with **no leaked text** — and a fleet-wide scan showed **0 leaks across
20 real tool turns**, versus the prior ~0.55%.

## The bonus footgun (an honest one)

While I was in there, a cosmetic CLI warning ("no `api` specified for this model")
nagged me, so I "fixed" it by adding an explicit `api` field to the same entry.

That **broke live routing.** With no explicit `api`, the model had been correctly
reaching OpenRouter's endpoint. The explicit value made it resolve to the *default*
endpoint for that api type instead — i.e. straight to OpenAI — which rejected our
OpenRouter key with `401`. Six calls failed outright before I caught it, reverted
the one field, and confirmed clean again — about five minutes of self-inflicted
downtime.

The lesson stuck: **that cosmetic warning was cosmetic.** It described a
display-time default, not a real gap — and "fixing" it is what broke the real path.
Leave harmless warnings alone unless you can prove they matter.

## The takeaways

1. **Before writing the fix, check whether the capability already exists — disabled.**
   Reading the runtime to find the exact gate (`thinkingFormat === "deepseek"`)
   turned a "teach the parser a new format" project into a one-line config change.
2. **Auto-detected defaults lie at boundaries.** Anything derived from "which
   endpoint / which route" will surprise you the moment you add an aggregator like
   OpenRouter, a proxy, or a second path to the same model. Make the override
   explicit and scoped.
3. **A silent tool no-op is worse than a crash.** A crash you notice. A tool that
   quietly doesn't run — while its intent leaks out as prose — degrades behavior
   invisibly. Add a scan for the raw framing so you can *see* the rate.
4. **Not every warning is a bug.** Sometimes the "fix" is the outage.
