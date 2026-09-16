# A fallback that can also fail is not a safety net

*Build-in-public notes on giving every agent a floor it can't fall through.*

## The trigger

Our primary model hit its usage cap for a few hours ("reached your subscription
limit, resets at 3pm"). No problem — we have a fallback. Everything shifted to a
cheaper cloud model and kept going, just slower.

But it made us ask the uncomfortable question: **what if the fallback is down
too?** Our chain for the core agents was `primary → cheap cloud → second cloud`.
All three live on the network. A bad enough provider outage — or a second
rate-limit — and those agents would have *nothing*. The pipeline wouldn't
degrade; it would stop.

## The principle

> A fallback that shares a failure mode with the thing it's backing up isn't a
> safety net. End the chain in something you fully control.

For us that means a **local model** as the last resort — running on our own
hardware, no rate limit, no provider outage, always reachable. It's weaker than
the cloud models, but "weaker and available" beats "excellent and offline." The
job of the last link isn't to be good; it's to **keep the agent alive** — able to
answer simply, or at least hand off to a coordinator — until the cloud comes back.

So the chain becomes:

```yaml
model:
  primary:  cloud/best            # quality, normal operation
  fallbacks:
    - cloud/cheap                 # cost-effective first fallback
    - cloud/second-provider       # different provider = independent failure
    - local/small                 # last resort: always available, no rate limit
```

The local model only ever fires if **all three** cloud options fail — so normal
behavior is completely unchanged. It's pure insurance.

## The gotcha that made this necessary

Here's the part that bit us. We *did* have a sensible default chain — the shared
default already ended in local models. But several of our most important agents
had their own **explicit per-agent model config**, and an explicit config
**replaces** the default, it doesn't extend it. So those agents had quietly
inherited only `[cheap, second]` — no local floor — precisely because someone had
customized them earlier for other reasons.

> An explicit override silently drops whatever it didn't restate.

The eight agents most likely to be customized were exactly the eight missing the
safety net. Worth auditing for in any config system with "defaults + overrides."

## Two trade-offs worth stating out loud

1. **Order fallbacks by cost, floor them by capability.** Cheapest-that-still-works
   goes first (you'll spend most of your fallback time there); the local model goes
   last (capability floor, not first choice).

2. **Size the last resort for the *worst* case, not the average.** If a broad cloud
   outage makes *many* agents fall to local at the same moment, they contend for the
   same GPU. We deliberately chose a **small** local model over our largest one for
   the last-resort slot, so a simultaneous fallback degrades gracefully instead of
   starving the GPU (which also runs an image pipeline). The last link should be
   *light and reliable*, not big and greedy.

## What changed

- Every core agent now ends its chain in a local model. During a provider
  slowdown or outage, nothing goes fully dark — worst case, agents run degraded on
  local hardware and recover automatically when the cloud returns.
- Normal operation is untouched (primary still runs everything; the floor never
  fires unless it's genuinely needed).
- We audited every agent's *effective* chain, not just the default, because the
  override-drops-the-rest trap is invisible until you print the resolved config.

## The takeaways

1. **Your last fallback must not share a failure mode with the rest.** Local, or
   at least a fully independent provider you control the quota on.
2. **Print the *resolved* config, not the template.** Overrides hide gaps; the only
   truth is what each agent actually ends up with.
3. **Insurance you never trigger is still worth buying.** The floor added zero cost
   to normal operation and removed a whole category of "the outage took everything
   down" incidents.
