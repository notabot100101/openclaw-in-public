# A slow agent turn is not a failed turn

*Build-in-public notes from wiring up a reliable multi-agent pipeline.*

## The symptom

We run an email assistant. Routine mail is handled by a lightweight "mail
worker"; anything ambiguous or multi-step gets handed off to a heavier
"coordinator" agent that can plan, search memory, and delegate. The worker
invokes the coordinator as a subprocess and waits for its answer, then emails it
back.

One morning the owner got three of these instead of an answer:

> *"I tried to get you a complete answer 3 times and it kept failing internally,
> so I don't have a reliable answer to give you here."*

The thing is — the questions were easy. And the "failure" email was itself proof
the pipeline was alive. So what was actually failing?

## The dig

Two facts, found by reading transcripts instead of guessing:

1. **The primary model was rate-limited.** Every coordinator turn was falling
   through to slower backup models. A turn that normally takes seconds was taking
   **~12 minutes**.

2. **The worker killed the subprocess at 5 minutes.** It launched the coordinator
   with a 300-second timeout. When that elapsed, it killed the process, and —
   because it had no way to recover the session's id after a `SIGKILL` — it
   returned `(failed, no-answer, no-session-id)`. Three of those in a row tripped
   the "give up and tell the user" path.

Here's the kicker. The runtime **kept executing the turn server-side** after the
CLI was killed (its own agent timeout was far longer than 5 minutes). Both killed
sessions **finished, with correct, complete answers**, sitting in the transcript
files — which the worker had already given up on and thrown away.

So the pipeline wasn't broken. It was **impatient**. It was scoring *slow* as
*failed*, and discarding good work.

## The insight

> A turn that is still running is not a turn that has failed.

The kill severed our *handle* to the work, not the work itself. If we could keep
a handle across the timeout, a slow turn could simply be **deferred** — read its
answer on a later pass — exactly the way we already handle a coordinator that
*cooperatively* yields to a sub-task and finishes later. We had that "read it next
run" machinery already; timeouts just weren't routed into it.

## The fix

Two small changes:

1. **Self-assign the session id up front.** Instead of letting the runtime mint an
   id we can only learn from the process's final output (which a kill destroys),
   we generate the id ourselves and pass it in. Now the transcript path is known
   *before* the call — so it's still known *after* a kill.

2. **On timeout, defer instead of fail.** Catch the timeout specifically and
   return the known id, which drops the item into the existing "poll briefly, else
   read next run" path — and, crucially, **does not advance the give-up counter**.
   Genuine errors (non-zero exit, real exceptions) still fail hard, so a truly
   dead turn still escalates.

```python
# Self-assign the id so a killed subprocess still leaves a readable breadcrumb.
handoff_id = f"handoff-{utcnow_compact_micros()}"

try:
    result = run_subprocess(
        ["agent", "--session-id", handoff_id, "--message", instruction, "--json"],
        timeout=CLI_WAIT_SECONDS,          # short: we don't want to block the batch
    )
    # ... normal success path: parse and return the answer ...

except SubprocessTimeout:
    # NOT a failure: the runtime is still finishing this turn server-side.
    # Hand back the known id so the caller defers and reads the completed
    # answer on the next pass — same path as a cooperative yield.
    log(f"handoff exceeded {CLI_WAIT_SECONDS}s CLI wait; deferring to read "
        f"completion next run (session {handoff_id})")
    return Deferred(session_id=handoff_id)

except Exception as e:
    # A real error (bad exit code, crash) still fails hard and escalates.
    log(f"handoff error: {e}")
    return Failed()
```

The caller already knew what to do with a "deferred" result: keep the id, and on
the next pass read the transcript at `sessions/<id>` — which by then holds the
finished answer — and send it.

## What changed for the user

- During a provider slowdown, they now get the **real answer a cycle later**
  instead of a false "I couldn't answer."
- The give-up email only fires on **actual** failures — a crash, or a session that
  genuinely never produces output — not on slowness.
- We verified it three ways: a live probe proving a self-assigned id creates a
  readable transcript; unit tests that a timeout returns a usable id while a
  non-timeout error still returns nothing; and confirmation that the two sessions
  which had triggered the original false failures held complete, deliverable
  answers the old code had discarded.

## The takeaways

1. **Match the log line to the code path before theorizing.** The failure text
   ("give up after 3") pointed at the retry counter, not at the model — which is
   where the real cause was.
2. **Distinguish "no result yet" from "no result."** A timeout is the former. If
   your system can't tell them apart, it will throw away good work under load.
3. **Keep a handle you control.** Self-assigning the work id turned an
   unrecoverable kill into a resumable, readable deferral — a tiny change that
   removed a whole class of false failures.

*Slow is a capacity problem. Failed is a correctness problem. Don't let your
retry logic confuse the two.*
