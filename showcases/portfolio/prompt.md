Build me a plain-text (no curses, no raw terminal mode) terminal client for
my personal income-portfolio web app's own JSON API, designed to be typed
entirely from an SSH session -- including from a mobile browser terminal,
where arrow keys and escape sequences aren't reliable. Every control should
be a letter or number typed then Enter, nothing raw-mode.

1. A live-updating Overview screen as the home/landing screen: total value
   (including a manual 401k entry), a breakdown by account, cash, and
   income metrics (yield on cost, current yield, forward income per
   year/month/day, trailing-twelve-months income). Below the numbers, a
   line chart of total portfolio value over a selectable timeframe (1D/1W/
   1M/1Y), rendered as text -- Unicode braille characters packed into a
   2x4 sub-pixel grid per character cell, so it's real chart resolution
   using only stdlib/unicode, no plotting library, and it prints inline
   like any other line of terminal output over plain SSH.

2. The Overview should silently poll the API in the background on a fixed
   interval and redraw in place if I haven't typed anything -- implemented
   with a select()-based read-with-timeout on stdin rather than switching
   the terminal into raw/curses mode, so typing still works exactly like a
   normal blocking input() the moment I start typing. A failed poll should
   just skip a beat, not crash or show an error.

3. Every other screen is reached by typing a single digit from the
   Overview and returns to Overview on "q" -- positions table, a
   hypothetical-401k tracker (view + update balance/allocation, posts back
   to the API), income milestones with progress bars and ETA estimates,
   recent distributions, recent contributions, market news, a colored
   sector/index heatmap (background-color intensity scaled by daily
   percent move, clamped so a handful of big movers don't wash out the
   rest of the grid), and a ticker search that reuses the same braille
   price chart with its own selectable range.

4. Auth to the API is a simple session -- POST a shared PIN to a login
   endpoint, keep the resulting cookie in an in-memory cookie jar for the
   life of the process, reuse it for every subsequent request.

5. Cosmetic details: a figlet+lolcat startup banner that measures the
   actual rendered width of a few candidate fonts against the real
   terminal width and picks the widest one that still fits (falling back
   to plain colored text if even the smallest font would wrap), and ANSI
   green/red coloring for gains/losses throughout.

The whole point is a full-featured live dashboard for a personal finance
web app that's comfortable to actually use from a phone's SSH client, not
just a technically-possible fallback for when a browser isn't handy.
