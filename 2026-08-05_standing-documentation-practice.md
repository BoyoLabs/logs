# Establishing a Standing Documentation Practice - 2026-08-05

## Context & Intent
This entry is about itself, in a sense: it documents the decision to make
documentation of computer-scientific work a *default*, proactive behavior rather
than something requested case by case.

The reasoning that triggered it: markdown files are cheap. Text is small enough,
storage and compute cost is negligible enough, that there's no real tradeoff being
made by writing things down generously. If it costs almost nothing and it fixes a
real gap (work happening and then not being captured anywhere durable), there's no
good argument against doing it by default.

## The Execution

1. **Real git clones, replacing ad-hoc staging.** `/home/boyo/Desktop/logs` had been
   a local staging folder for the `logs` repo, populated by a prior workflow of
   writing files locally and uploading them to GitHub by hand (`git log` on the
   `logs` repo shows a literal `Add files via upload` commit from that era). Cloned
   a proper working copy to `/home/boyo/logs` instead, matching the git-push
   workflow already established for `prompt-repo` at `/home/boyo/prompt-repo`
   earlier in the session. The old staging folder was left alone, not deleted, but
   is no longer the active path.

2. **Routing logic formalized.** Two repos now have a clear division:
   - `BoyoLabs/logs` — investigations, debugging, fixes, experiments. Negative
     results explicitly included, per that repo's own stated philosophy.
   - `BoyoLabs/prompt-repo` — AI-built things (mostly `*.boyolabstech.com`
     services) paired with the reproducible master prompt that built them.
   Some work can produce entries in both — a build gets a `prompt-repo` write-up,
   the debugging/decisions around building it gets a `logs` entry.

3. **Persisted as a standing instruction**, not just a one-off conversation. Saved
   to Claude's cross-session memory so the behavior survives past this single chat:
   proactively write the doc as part of finishing substantive technical work,
   without waiting to be asked, the same way a summary or a test run would
   naturally be part of finishing something.

## Findings

The gap this closes: prior to today, documentation only happened when explicitly
requested (as with this repo's own README, and most of the older log entries in
this very folder). That meant most of the actual thinking and problem-solving that
happens in a session evaporated the moment the session ended, unless someone
remembered to ask for a writeup afterward — usually after the interesting details
had already gone stale in memory.

The fix is not a new tool or process, just a lowered bar: since the cost of writing
a markdown file is close to zero, the default should be "write it down" rather than
"write it down if asked." This log is the first artifact produced under that
default rather than by request — proof the practice is live, not just declared.
