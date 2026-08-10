# A Plain-Text Terminal Dashboard for a Personal Income-Portfolio App

## What it builds

`portfolio` is a full-featured terminal client for the JSON API behind a
personal income-investing web app — live totals, a chart, positions, a
401k tracker, income milestones, distributions/contributions history,
market news, a sector/index heatmap, and ticker search/quotes, all reached
from one live-updating Overview screen.

Like the [boyoapps](../boyoapps/README.md) entry, the core constraint was
usability while remoted in — every control is a letter or number typed
then Enter, no arrow keys or raw terminal mode, because this is meant to
be genuinely comfortable to use from a phone's SSH client, not just a
technically-possible fallback for when a browser isn't handy.

## Design goals for remote/mobile use specifically

- **No curses, no raw terminal mode at all.** Unlike the boyoapps editor,
  this app deliberately stays in plain line-buffered ("cooked") mode —
  every input is a normal `input()`-style read-a-line-then-Enter, which
  survives inconsistent key-code mapping on mobile SSH clients far better
  than arrow-key/escape-sequence handling would.
- **Live background refresh without switching to raw mode.** The Overview
  screen polls the API on a timer and redraws in place if nothing's been
  typed — but implemented as a `select()`-based read-with-timeout on
  stdin, not a curses event loop. `select()` on a real terminal only
  reports readable once a full line has been typed and Enter hit, so this
  doesn't change the interaction model at all, it just allows a redraw
  while genuinely waiting for input.
- **Every chart is Unicode braille characters, not a plotting library.**
  Braille Unicode cells each pack a 2x4 sub-pixel grid, so a small
  character block renders at roughly 4x the resolution plain block
  characters would — real line-chart resolution using only the standard
  library and terminal Unicode support, no curses, no dependency that
  might not be installed.
- **A single flat, numbered menu** (2–9, with 1D/1W/1M/1Y as single
  letters) reachable from the Overview screen, rather than nested menus —
  minimizes how many keystrokes/screens deep any feature is from a
  small phone keyboard.

## Source

One file, `main.py` — no external Python packages; `figlet`/`lolcat` are
optional shell-level niceties for the startup banner. Two things are
redacted from this published copy that aren't in the private original:
the shared PIN used to authenticate to the API, and literal mentions of
the web app's own hostname (referred to here by role instead, per this
repo's standing practice — see the root README).

```python
#!/usr/bin/env python3
"""Terminal client for a personal income-portfolio web app's real
portfolio + 401k data. Overview is the home screen (live totals + chart);
every other screen is a number typed from there, and q on any of them
returns to Overview -- q on Overview itself quits. No curses. Every
control is a letter/number typed then Enter (no arrow keys, no raw
terminal mode) since this runs over an SSH session, often from a mobile
browser terminal where arrow/escape keys aren't reliable."""
import datetime
import http.cookiejar
import json
import os
import select
import shutil
import subprocess
import sys
import time
import urllib.error
import urllib.parse
import urllib.request

BASE_URL = "http://127.0.0.1:5080"
PIN = "[REDACTED-PIN]"

TIMEFRAMES = [("d", "1D", 1), ("w", "1W", 7), ("m", "1M", 30), ("y", "1Y", 365)]
REFRESH_SECONDS = 60  # matches the web app's own poll interval

GREEN = "\033[32m"
RED = "\033[31m"
CYAN = "\033[36m"
WHITE = "\033[97m"
BOLD = "\033[1m"
DIM = "\033[2m"
RESET = "\033[0m"

CHART_RANGES = ["1d", "1w", "1m", "3m", "ytd", "1y", "5y", "all"]
CHART_RANGE_LABELS = {"1d": "1D", "1w": "1W", "1m": "1M", "3m": "3M", "ytd": "YTD", "1y": "1Y", "5y": "5Y", "all": "ALL"}
HEATMAP_CLAMP = 0.025  # same clamp the web app uses for tile-color intensity


def api_session():
    jar = http.cookiejar.CookieJar()
    opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
    req = urllib.request.Request(
        f"{BASE_URL}/login",
        data=b"pin=" + PIN.encode(),
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    opener.open(req).read()
    return opener


def fetch_overview():
    opener = api_session()
    with opener.open(f"{BASE_URL}/api/cli-overview") as resp:
        return json.load(resp)


def fetch_news():
    opener = api_session()
    with opener.open(f"{BASE_URL}/api/cli-news") as resp:
        return json.load(resp)


def fetch_chart_history():
    opener = api_session()
    with opener.open(f"{BASE_URL}/api/chart-history") as resp:
        return json.load(resp)["points"]


def fetch_summary():
    """Lightweight headline-numbers endpoint -- no live requote of every
    position, unlike cli-overview. This is what the website itself polls
    every REFRESH_SECONDS for its live-updating totals, so we do the same
    rather than hammering the heavier full-refresh endpoint on a timer."""
    opener = api_session()
    with opener.open(f"{BASE_URL}/api/portfolio-summary") as resp:
        return json.load(resp)


def fetch_heatmap():
    opener = api_session()
    with opener.open(f"{BASE_URL}/api/cli-heatmap") as resp:
        return json.load(resp)


def fetch_search(query):
    opener = api_session()
    url = f"{BASE_URL}/api/search?q={urllib.parse.quote(query)}"
    with opener.open(url) as resp:
        return json.load(resp)["results"]


def fetch_quote(symbol):
    opener = api_session()
    try:
        with opener.open(f"{BASE_URL}/api/quote/{urllib.parse.quote(symbol)}") as resp:
            return json.load(resp)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise


def fetch_ticker_chart(symbol, range_key):
    opener = api_session()
    url = f"{BASE_URL}/api/chart/{urllib.parse.quote(symbol)}?range={range_key}"
    with opener.open(url) as resp:
        return json.load(resp)["points"]


def post_401k_update(balance, jepi_pct):
    opener = api_session()
    body = urllib.parse.urlencode({"balance": balance, "jepi_pct": jepi_pct}).encode()
    req = urllib.request.Request(
        f"{BASE_URL}/401k/update",
        data=body,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    opener.open(req).read()


def clear():
    os.system("clear")


FIGLET_FONTS = ["slant", "small", "mini"]


def _figlet(name, font):
    try:
        fig = subprocess.run(["figlet", "-f", font, name], capture_output=True, text=True, timeout=2)
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return None
    if fig.returncode != 0 or not fig.stdout.strip():
        return None
    return fig.stdout


def print_banner(name):
    """Same figlet+lolcat house style as the ssh shell rice (see `clear` in
    .bashrc), but width-checked first -- that banner just wraps mid-glyph
    on a narrow mobile terminal, so this measures the actual rendered width
    per font against the real terminal and picks the widest one that
    fits, falling back to plain colored text if even the smallest doesn't."""
    if not sys.stdout.isatty():
        return
    term_w = shutil.get_terminal_size(fallback=(80, 24)).columns

    art = None
    for font in FIGLET_FONTS:
        candidate = _figlet(name, font)
        if candidate is None:
            continue
        widest = max((len(line) for line in candidate.splitlines()), default=0)
        if widest <= term_w:
            art = candidate
            break

    if art is None:
        print(f"{BOLD}{CYAN}{name}{RESET}")
        time.sleep(0.2)
        return

    try:
        lol = subprocess.run(["lolcat", "-f"], input=art, capture_output=True, text=True, timeout=2)
        print(lol.stdout if lol.returncode == 0 else art, end="")
    except (FileNotFoundError, subprocess.TimeoutExpired):
        print(art, end="")
    time.sleep(0.3)


def color_change(dc):
    if not dc:
        return ""
    c = GREEN if dc["cls"] == "pos" else RED
    return f"{c}{dc['arrow']} {dc['text']}{RESET}"


def _parse_iso(t):
    return datetime.datetime.fromisoformat(t.replace("Z", "+00:00"))


_BRAILLE_DOT = {
    (0, 0): 0x01, (0, 1): 0x02, (0, 2): 0x04, (0, 3): 0x40,
    (1, 0): 0x08, (1, 1): 0x10, (1, 2): 0x20, (1, 3): 0x80,
}


def braille_chart(values, width, height):
    """Renders `values` as a line chart using Unicode braille cells, each
    packing a 2x4 sub-pixel grid -- so a `width`x`height` character block
    gets 4x the resolution of plain block characters. Pure stdlib/unicode,
    no curses or external plotting lib, so it prints like any other line
    in this app and works over plain ssh."""
    if len(values) < 2:
        return None
    lo, hi = min(values), max(values)
    span = (hi - lo) or 1.0
    px_w, px_h = width * 2, height * 4
    n = len(values)

    def sample(i):
        pos = i * (n - 1) / (px_w - 1) if px_w > 1 else 0
        lo_i = int(pos)
        hi_i = min(lo_i + 1, n - 1)
        frac = pos - lo_i
        return values[lo_i] * (1 - frac) + values[hi_i] * frac

    def to_row(v):
        r = round((v - lo) / span * (px_h - 1))
        return px_h - 1 - r

    rows = [to_row(sample(x)) for x in range(px_w)]
    grid = [[False] * px_w for _ in range(px_h)]
    for x in range(px_w):
        r = rows[x]
        r0 = rows[x - 1] if x else r
        step = 1 if r >= r0 else -1
        for rr in range(r0, r + step, step):
            grid[rr][x] = True

    lines = []
    for cy in range(height):
        chars = []
        for cx in range(width):
            bits = 0
            for dy in range(4):
                for dx in range(2):
                    if grid[cy * 4 + dy][cx * 2 + dx]:
                        bits |= _BRAILLE_DOT[(dx, dy)]
            chars.append(chr(0x2800 + bits) if bits else " ")
        lines.append("".join(chars))
    return lines, lo, hi


def _fmt_axis_time(iso, same_day):
    dt = _parse_iso(iso)
    return dt.strftime("%H:%M") if same_day else dt.strftime("%b %d")


def render_chart(points, title_line, money_decimals=0):
    """Shared by the portfolio-value chart (Overview) and ticker-price
    charts (Search) -- points are already scoped to the desired range/
    timeframe by the caller, this just draws them."""
    if len(points) < 2:
        print(f"  {DIM}not enough data to chart{RESET}")
        return
    term_w = shutil.get_terminal_size(fallback=(80, 24)).columns
    width = max(20, min(term_w - 6, 60))
    height = 8
    values = [p["y"] for p in points]
    chart = braille_chart(values, width, height)
    if not chart:
        return
    lines, lo, hi = chart
    color = GREEN if values[-1] >= values[0] else RED

    print(title_line)
    for line in lines:
        print(f"  {color}{line}{RESET}")
    lo_s, hi_s = f"${lo:,.{money_decimals}f}", f"${hi:,.{money_decimals}f}"
    print(f"  {DIM}{lo_s}{' ' * max(1, width - len(lo_s) - len(hi_s))}{hi_s}{RESET}")
    same_day = _parse_iso(points[0]["t"]).date() == _parse_iso(points[-1]["t"]).date()
    start_l = _fmt_axis_time(points[0]["t"], same_day)
    end_l = _fmt_axis_time(points[-1]["t"], same_day)
    print(f"  {DIM}{start_l}{' ' * max(1, width - len(start_l) - len(end_l))}{end_l}{RESET}")


def show_value_chart(points, tf_idx):
    key, label, days = TIMEFRAMES[tf_idx]
    if not points:
        print(f"  {DIM}chart unavailable (couldn't reach server){RESET}")
        return
    cutoff = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=days)
    filtered = [p for p in points if _parse_iso(p["t"]) >= cutoff]
    if len(filtered) < 2:
        filtered = points[-2:]
    if len(filtered) < 2:
        print(f"  {DIM}not enough history yet -- check back soon{RESET}")
        return

    values = [p["y"] for p in filtered]
    pct = (values[-1] - values[0]) / values[0] * 100 if values[0] else 0
    arrow = "↑" if pct >= 0 else "↓"
    color = GREEN if pct >= 0 else RED
    title = f"  {BOLD}Total Portfolio Value{RESET}  {DIM}({label}){RESET}  {color}{arrow} {pct:+.2f}%{RESET}"
    render_chart(filtered, title)


def header(subtitle):
    clear()
    print(f"{BOLD}{CYAN}Portfolio Overview{RESET} -- {subtitle}\n")


def wait_back():
    while True:
        choice = input(f"\n{DIM}[q] back to overview{RESET} > ").strip().lower()
        if choice == "q":
            return


def _read_line_with_timeout(prompt, seconds):
    """Like input(), but gives up and returns None after `seconds` of no
    input. stdin stays in normal line-buffered (cooked) mode the whole
    time -- select() on a tty only reports readable once a full line has
    been typed and Enter hit, so this doesn't change the type-then-Enter
    interaction at all, it just lets us redraw while waiting."""
    print(prompt, end="", flush=True)
    ready, _, _ = select.select([sys.stdin], [], [], seconds)
    if not ready:
        return None
    line = sys.stdin.readline()
    if line == "":  # EOF (stdin closed)
        raise EOFError
    return line.strip().lower()


def show_overview(data):
    try:
        chart_points = fetch_chart_history()
    except urllib.error.URLError:
        chart_points = []

    tf_idx = 0  # 1D, matching the web app's default tab
    last_updated = datetime.datetime.now()
    while True:
        header(f"live -- last updated {last_updated.strftime('%H:%M:%S')}")
        t = data["totals"]
        print(f"  {BOLD}Total incl. 401k:{RESET}  {(data['total_incl_401k_fmt'] or '—'):<12} {color_change(data['total_daily'])}")
        print(f"  Brokerage:           {t['portfolio_value_fmt']:<12} {color_change(t['portfolio_daily'])}")
        if data["h401k"]:
            h = data["h401k"]
            print(f"  401k (hypothetical): {h['balance_fmt']:<12} {color_change(h['balance_daily'])}")
        print(f"  Cash:                {t['cash_fmt']}")
        print()
        print(f"  Yield on cost: {t['blended_yield_on_cost_fmt']}   Current yield: {t['blended_current_yield_fmt']}")
        print(f"  Forward income: {t['forward_income_fmt']}/yr   {t['forward_income_monthly_fmt']}/mo   {t['forward_income_daily_fmt']}/day")
        print(f"  TTM income:     {t['ttm_income_fmt']}/yr   {t['ttm_income_monthly_fmt']}/mo")
        print()
        show_value_chart(chart_points, tf_idx)

        print()
        print(f"  {DIM}jump to:{RESET}")
        for key, label, _ in MENU:
            print(f"  {key}) {label}")

        tf_keys = "  ".join(f"[{k}] {lbl}" for k, lbl, _ in TIMEFRAMES)
        prompt = f"\n  {tf_keys}   {DIM}[2-9] jump   [q] quit{RESET} > "
        choice = _read_line_with_timeout(prompt, REFRESH_SECONDS)

        if choice is None:
            # Timed out with nothing typed -- silent background refresh,
            # same as the web app's own poll-and-redraw-in-place. A
            # failed poll (network hiccup) just skips a beat, matching
            # the web app's behavior on a failed fetch.
            try:
                chart_points = fetch_chart_history() or chart_points
                summary = fetch_summary()
                t["portfolio_value_fmt"] = summary["portfolio_value_fmt"]
                t["portfolio_daily"] = summary["portfolio_daily"]
                if summary.get("total_portfolio_value_fmt"):
                    data["total_incl_401k_fmt"] = summary["total_portfolio_value_fmt"]
                data["total_daily"] = summary["total_daily"]
                last_updated = datetime.datetime.now()
            except urllib.error.URLError:
                pass
            continue

        if choice == "q":
            return data
        matched_tf = False
        for i, (k, _, _) in enumerate(TIMEFRAMES):
            if choice == k:
                tf_idx = i
                matched_tf = True
                break
        if matched_tf:
            continue
        for key, _, fn in MENU:
            if choice == key:
                data = fn(data)
                break


def show_positions(data):
    header("brokerage positions")
    cols = ["TICKER", "ACCT", "SHARES", "PRICE", "VALUE", "COST", "P/L", "YOC", "CY"]
    widths = [7, 11, 12, 9, 11, 11, 11, 8, 8]
    print("  " + "".join(c.ljust(w) for c, w in zip(cols, widths)))
    for p in data["positions"]:
        pl_color = GREEN if p["positive"] else RED
        line = "  "
        line += p["ticker"].ljust(widths[0])
        line += p["account"].ljust(widths[1])
        line += p["shares_fmt"].ljust(widths[2])
        line += p["price_fmt"].ljust(widths[3])
        line += p["value_fmt"].ljust(widths[4])
        line += p["cost_basis_fmt"].ljust(widths[5])
        line += f"{pl_color}{p['pl_fmt']:<{widths[6]}}{RESET}"
        line += p["yield_on_cost_fmt"].ljust(widths[7])
        line += p["current_yield_fmt"].ljust(widths[8])
        print(line)
    wait_back()
    return data


def update_401k_flow(data):
    h = data["h401k"]
    current_jepi_pct = round((h["legs"][0]["pct"] if h else 0.5) * 100, 2)
    print()
    balance_str = input(f"  New balance (current {h['balance_fmt'] if h else '—'}), blank to cancel: $").strip()
    if not balance_str:
        print("  Cancelled.")
        input(f"\n{DIM}[enter] continue{RESET} > ")
        return data
    try:
        balance = float(balance_str.replace(",", "").replace("$", ""))
    except ValueError:
        print("  Not a number, cancelled.")
        input(f"\n{DIM}[enter] continue{RESET} > ")
        return data

    jepi_str = input(f"  JEPI % [{current_jepi_pct:g}]: ").strip()
    try:
        jepi_pct = float(jepi_str) if jepi_str else current_jepi_pct
    except ValueError:
        print("  Not a number, cancelled.")
        input(f"\n{DIM}[enter] continue{RESET} > ")
        return data

    try:
        post_401k_update(balance, jepi_pct)
        print("  Updated. Refreshing...")
        data = fetch_overview()
    except urllib.error.URLError as e:
        print(f"  Update/refresh failed: {e}")
        input(f"\n{DIM}[enter] continue{RESET} > ")
    return data


def show_401k(data):
    while True:
        header("401k (hypothetical JEPI/JEPQ split)")
        h = data["h401k"]
        if not h:
            print("  No 401k data on file.")
        else:
            print(f"  Balance:  {h['balance_fmt']}  {color_change(h['balance_daily'])}")
            print(f"  Blended current yield: {h['blended_current_yield_fmt']}")
            print(f"  Forward income: {h['forward_income_monthly_fmt']}/mo")
            print(f"  Updated: {h['updated_at'][:10]}")
            print()
            print("  " + "TICKER".ljust(8) + "ALLOC %".ljust(10) + "ALLOC $".ljust(12) + "SHARES".ljust(12) + "FWD INCOME/yr")
            for leg in h["legs"]:
                print(
                    "  " + leg["ticker"].ljust(8) + leg["pct_fmt"].ljust(10)
                    + leg["alloc_fmt"].ljust(12) + leg["shares_fmt"].ljust(12) + leg["forward_income_fmt"]
                )
        choice = input(f"\n{DIM}[u] update balance   [q] back to overview{RESET} > ").strip().lower()
        if choice == "q":
            return data
        if choice == "u":
            data = update_401k_flow(data)


def show_milestones(data):
    header("income milestones")
    bar_width = 24
    for m in data["milestones"]:
        filled = round(min(100, m["pct"]) / 100 * bar_width)
        bar = "#" * filled + "-" * (bar_width - filled)
        tag = " (incl. 401k)" if m["includes_401k"] else ""
        print(f"  {m['name']}{tag}")
        print(f"    [{bar}] {m['pct']}%   {m['value_fmt']} / {m['target_fmt']}")
        if m["reached"]:
            print(f"    {GREEN}reached{RESET}")
        elif m["months_fmt"]:
            print(f"    ETA {m['eta_fmt']} ({m['months_fmt']})")
        else:
            print("    ETA unknown (no income growth assumed at this pace)")
        print()
    wait_back()
    return data


def show_distributions(data):
    header("recent distributions (last 10)")
    for d in data["distributions"]:
        print(f"  {d['pay_date']}  {d['ticker']:<6} {d['total_fmt']:<10} sgov {d['sgov_fmt']:<10} accum {d['accum_fmt']}")
        if d["notes"]:
            print(f"    {DIM}{d['notes']}{RESET}")
    if not data["distributions"]:
        print("  None on file.")
    wait_back()
    return data


def show_contributions(data):
    header("recent contributions (last 10)")
    for c in data["contributions"]:
        print(f"  {c['date']}  {c['ticker']:<6} {c['amount_fmt']:<10} {c['shares_fmt']:<12} @ {c['price_fmt']}")
        if c["notes"]:
            print(f"    {DIM}{c['notes']}{RESET}")
    if not data["contributions"]:
        print("  None on file.")
    wait_back()
    return data


def show_news(data):
    header("market news")
    try:
        news = fetch_news()
    except urllib.error.URLError as e:
        print(f"  Couldn't fetch news: {e}")
        wait_back()
        return data
    items = news["items"][:10]
    if not items:
        print("  None available.")
    for it in items:
        print(f"  {BOLD}{it['title']}{RESET}")
        print(f"    {DIM}{it['source']} -- {it['published_fmt']}{RESET}")
        if it["summary"]:
            print(f"    {it['summary']}")
        print(f"    {DIM}{it['link']}{RESET}")
        print()
    wait_back()
    return data


def _heatmap_bg(pct):
    if pct is None:
        r, g, b = 58, 58, 58
    else:
        intensity = min(abs(pct), HEATMAP_CLAMP) / HEATMAP_CLAMP
        if pct >= 0:
            r, g, b = round(28 + intensity * 20), round(90 + intensity * 140), round(60 + intensity * 30)
        else:
            r, g, b = round(110 + intensity * 140), round(42 + intensity * 20), round(50 + intensity * 20)
    return f"\033[48;2;{r};{g};{b}m"


def _heatmap_row(m, width, indent=""):
    bg = _heatmap_bg(m["pct"])
    fixed = len(indent) + 7 + 10 + 9 + 2  # ticker + price + pct + trailing pad
    name_w = max(4, width - fixed)
    name = m["name"][:name_w]
    text = f"{indent}{m['ticker']:<7}{name:<{name_w}}{m['price_fmt']:>10}{m['pct_fmt']:>9}  "
    text = text[:width].ljust(width)
    return f"{bg}{BOLD}{WHITE}{text}{RESET}"


def show_heatmap(data):
    try:
        hm = fetch_heatmap()
    except urllib.error.URLError as e:
        header("heatmap")
        print(f"  Couldn't fetch heatmap: {e}")
        wait_back()
        return data

    while True:
        term_w = shutil.get_terminal_size(fallback=(80, 24)).columns
        width = max(40, min(term_w - 4, 70))
        header(f"heatmap -- as of {hm['refreshed']}")
        print(f"  {BOLD}Major Indexes{RESET}")
        for g in hm["indexes"]:
            print(f"  {_heatmap_row(g['index'], width)}")
            for etf in g["etfs"]:
                print(f"  {_heatmap_row(etf, width, indent='  ')}")
        print()
        print(f"  {BOLD}Sectors{RESET}  {DIM}(best to worst){RESET}")
        for s in hm["sectors"]:
            print(f"  {_heatmap_row(s, width)}")

        choice = input(f"\n  {DIM}[r] refresh   [q] back to overview{RESET} > ").strip().lower()
        if choice == "q":
            return data
        if choice == "r":
            try:
                hm = fetch_heatmap()
            except urllib.error.URLError:
                pass


def show_quote(symbol):
    header(f"search -- {symbol}")
    try:
        q = fetch_quote(symbol)
    except urllib.error.URLError as e:
        print(f"  Couldn't fetch quote: {e}")
        wait_back()
        return
    if not q:
        print(f"  {DIM}No quote found for {symbol}.{RESET}")
        wait_back()
        return

    range_idx = 0
    while True:
        range_key = CHART_RANGES[range_idx]
        try:
            points = fetch_ticker_chart(q["symbol"], range_key)
        except urllib.error.URLError:
            points = []

        header(f"search -- {q['symbol']}")
        print(f"  {BOLD}{q['name']}{RESET}")
        print(f"  {DIM}{q['symbol']} -- {q['currency']}{RESET}")
        print()
        pct = q["pct"] or 0
        arrow = "↑" if pct >= 0 else "↓"
        color = GREEN if pct >= 0 else RED
        delta_txt = f"{q['delta']:+.2f}" if q["delta"] is not None else "—"
        pct_txt = f"{pct * 100:+.2f}%" if q["pct"] is not None else "—"
        print(f"  {BOLD}${q['price']:,.2f}{RESET}   {color}{arrow} {delta_txt} ({pct_txt}){RESET}")
        if q["extended_label"]:
            epct = q["extended_pct"] or 0
            earrow = "↑" if epct >= 0 else "↓"
            ecolor = GREEN if epct >= 0 else RED
            print(f"  {DIM}{q['extended_label']}:{RESET} ${q['extended_price']:,.2f}  {ecolor}{earrow} {q['extended_delta']:+.2f} ({epct * 100:+.2f}%){RESET}")
        print()
        if q["dividend_yield_pct"] is not None:
            print(f"  Dividend yield: {q['dividend_yield_pct'] * 100:.2f}%   {q['dividend_frequency']}   (${q['dividend_annual_per_share']:.2f}/yr)")
        else:
            print(f"  {DIM}{q['dividend_frequency']}{RESET}")
        print()

        if points:
            values = [p["y"] for p in points]
            cpct = (values[-1] - values[0]) / values[0] * 100 if values[0] else 0
            carrow = "↑" if cpct >= 0 else "↓"
            ccolor = GREEN if cpct >= 0 else RED
            title = f"  {DIM}({CHART_RANGE_LABELS[range_key]}){RESET}  {ccolor}{carrow} {cpct:+.2f}%{RESET}"
            render_chart(points, title, money_decimals=2)
        else:
            print(f"  {DIM}chart unavailable for this range{RESET}")

        range_keys_line = "  ".join(f"[{k}]" for k in CHART_RANGES)
        choice = input(f"\n  {range_keys_line}   {DIM}[q] back to results{RESET} > ").strip().lower()
        if choice == "q":
            return
        if choice in CHART_RANGES:
            range_idx = CHART_RANGES.index(choice)


def show_search(data):
    while True:
        header("search")
        query = input("  Ticker or company name (blank to cancel): ").strip()
        if not query:
            return data
        try:
            results = fetch_search(query)
        except urllib.error.URLError as e:
            print(f"  Search failed: {e}")
            input(f"\n{DIM}[enter] continue{RESET} > ")
            continue
        if not results:
            print(f"  {DIM}No results for \"{query}\".{RESET}")
            input(f"\n{DIM}[enter] continue{RESET} > ")
            continue

        while True:
            header(f'search results -- "{query}"')
            for i, r in enumerate(results, 1):
                print(f"  {i}) {BOLD}{r['symbol']:<8}{RESET} {r['name'][:38]:<38} {DIM}{r['exchange']} -- {r['type']}{RESET}")
            choice = input(f"\n  {DIM}[1-{len(results)}] view   [s] new search   [q] back to overview{RESET} > ").strip().lower()
            if choice == "q":
                return data
            if choice == "s":
                break
            if choice.isdigit() and 1 <= int(choice) <= len(results):
                show_quote(results[int(choice) - 1]["symbol"])


MENU = [
    ("2", "Positions", show_positions),
    ("3", "401k", show_401k),
    ("4", "Milestones", show_milestones),
    ("5", "Recent distributions", show_distributions),
    ("6", "Recent contributions", show_contributions),
    ("7", "News", show_news),
    ("8", "Heatmap", show_heatmap),
    ("9", "Search", show_search),
]


def main():
    print_banner("portfolio")
    try:
        data = fetch_overview()
    except urllib.error.URLError as e:
        print(f"portfolio: couldn't reach the portfolio server ({e})", file=sys.stderr)
        sys.exit(1)

    try:
        show_overview(data)
    except (KeyboardInterrupt, EOFError):
        print()


if __name__ == "__main__":
    main()
```

## Master Prompt

Paste this to an AI coding assistant (e.g. Claude) to have it build a
similar client against your own personal-finance web app's API:

```
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
```

## Notes

- Built with Claude Code, against an existing Flask backend of my own.
- No external Python packages required to run the client itself —
  `figlet`/`lolcat` are optional shell-level niceties for the startup
  banner and degrade gracefully if absent.
- The PIN and the web app's hostname are redacted/generalized in this
  published copy — see the top of the Source section above.
