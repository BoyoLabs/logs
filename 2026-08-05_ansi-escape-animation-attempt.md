# Live ASCII Animation via Claude Code Chat Stream - 2026-08-05

## Context & Intent
Started from a broader question: what's the actual benefit of building a dedicated
AI-native application versus just asking Claude to do the thing inline? To make the
question concrete, we picked a stress-test case — animated ASCII art — and asked
whether Claude could generate a *live-animating* ASCII graphic directly inside a
Claude Code chat session (SSH'd into the box), with Claude itself as the
"interpreter" and no external script or tool call involved.

## Hypothesis
Claude Code streams assistant responses token-by-token into the terminal. If that
stream is passed to the terminal largely unprocessed, then embedding raw ANSI/VT100
control sequences (`ESC[2J` clear-screen, `ESC[H` cursor-home, etc.) directly in the
response text — outside of a code fence — could cause the terminal emulator to
interpret them live as the response streams. That would mean each new "frame"
overwrites the last in place, producing genuine redraw-style animation with zero
tooling: the chat response text itself doubling as a terminal control channel.

Counter-signal going in: the harness's own docs describe assistant text as being
"rendered in a monospace font using the CommonMark specification" — i.e., a
markdown-rendering layer sits between what Claude emits and what hits the terminal.
That layer could plausibly sanitize or strip control characters before they'd ever
reach a real VT100 interpreter.

## The Execution

**Attempt 1 — plain flipbook (no escape codes).** Printed a sequence of ASCII
frames (a dot bouncing across a corridor) as consecutive plain-text/code blocks with
no control characters, just to establish the baseline: sequential frames scroll by,
read as an animation only in the "flip the pages quickly" sense. Confirmed working,
as expected — no real redraw, just scroll.

**Attempt 2 — embedded raw escape sequences.** Rebuilt the same bouncing-dot
animation, but prefixed each frame with the literal escape byte (`0x1B`) followed by
`[2J[H` (clear screen + cursor home), placing the frame content in a fenced code
block below each escape prefix:

```
<ESC>[2J[H
[■                    ]

<ESC>[2J[H
[   ■                 ]

<ESC>[2J[H
[      ■              ]
...
```

Sent this live in the response stream and asked for a direct report of what actually
rendered on the terminal — no way to observe the rendered output from the model
side, so this had to be an empirical check via the human's eyes.

## Findings

The result: frames displayed as a scrolling flipbook (same as Attempt 1), **but**
with the literal text `[2J[H` printed above every frame — i.e., the escape *byte*
(`0x1B`) was stripped somewhere in the pipeline, while the rest of the sequence
(`[2J[H`, plain printable characters) survived and rendered as literal text.

This confirms the counter-signal from the hypothesis: the CommonMark/markdown
rendering layer between Claude's output and the terminal sanitizes control
characters before the terminal emulator ever sees them. The response stream is not
a raw passthrough to the terminal, regardless of how tight the perceived "typing"
connection feels — it's a parsed and rendered document. This rules out "smuggle
ANSI codes in chat text" as a way to get real in-place redraw animation inside a
Claude Code chat session, at least on the current rendering path.

## Conclusion / What Actually Works

Two viable paths remain for genuine (non-scrolling) animation:

1. **HTML/CSS/JS artifact** — publish an ASCII-styled page (monospace,
   terminal-green-on-black) where the animation is driven by real CSS/JS in a
   browser. Guaranteed to work since it isn't constrained by the chat rendering
   layer at all.
2. **A script executed directly in the terminal** (Python/bash using real
   `\x1b[2J`/`\x1b[H` escapes plus `sleep`) — genuine redraw-in-place, but it's a
   file the human runs themselves in their own shell, not something happening live
   "inside" the chat.

Net takeaway for the original AI-native-app question: this is itself a small data
point in favor of dedicated tooling over ad-hoc chat — the chat medium has real,
non-obvious rendering constraints (markdown sanitization) that a purpose-built
artifact or script simply doesn't have to fight against.
