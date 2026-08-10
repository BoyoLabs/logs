# A Curses-Based Python IDE, Built Into a Terminal App Launcher

## What it builds

`boyoapps` is a bare-word terminal app launcher (`~/bin/<command>` symlinks,
typed by name over SSH) that doubles as a scaffolding tool: it can create,
edit, and delete the small personal Python programs registered in it —
including running a purpose-built, no-external-dependency code editor
in-menu.

The core design goal is usability while remoted in — the whole flow (create
a new app, write it, save it, run it) has to work cleanly from a phone SSH
client on a narrow, laggy terminal, not just a full desktop terminal
emulator. That constraint shaped almost every implementation decision below.

- **`n`** — prompts for a command name + one-line description, scaffolds
  `<name>/main.py` from a template, registers it, symlinks it into `~/bin/`,
  and drops straight into the editor.
- **`e`** — opens the built-in editor on any app it scaffolded.
- **`d`** — deletes a registered app (dir, symlink, registry entry), with
  confirmation.
- On every save, it re-scans the script's top-level imports and reconciles
  a per-app virtualenv to match — installing anything third-party and
  repointing the shebang at the venv interpreter — so a generated app's
  dependencies are already satisfied the first time it's run standalone.

## Design goals for remote/mobile use specifically

A few decisions exist *only* because this has to work well from an SSH
session on a phone, not a full terminal:

- **Text entry during app creation is backspace-only, no arrow-key
  editing.** Deliberately dumbed down from what the full editor supports,
  because a single-line prompt doesn't need cursor movement and every extra
  keybinding is another thing that might not survive a mobile SSH
  keyboard's key-code mapping.
- **^S/^Q are explicitly un-hijacked from the terminal driver.** Those are
  XON/XOFF flow-control characters at the tty layer — without disabling
  that, the terminal silently eats them to pause/resume output and ^S
  (save) never reaches the program at all. Same fix real terminal editors
  like nano apply; found by testing, not by reading about it in advance.
- **The line-number gutter disappears under 50 columns** rather than
  cramming a narrow phone terminal further. Every screen write is wrapped
  in a try/except around `curses.error`, because writing to a terminal's
  bottom-right cell is a real ncurses crash if unguarded — it took down an
  earlier app in this family before every write got wrapped centrally.
- **Syntax highlighting uses only the Python standard library**
  (`tokenize` + `keyword`), not a package like `pygments`. A generated
  app's first save is often before any venv exists yet, so the editor
  itself can't depend on anything that needs installing first.
- **Auto-indent and a persistent status/help line** so it behaves close
  enough to a familiar editor (indent continues after a line ending in
  `:`, current line/total always visible) without trying to be a general
  text editor — it only ever edits one `main.py` at a time, from inside
  the create/edit flow.

## Source

Four files, no third-party imports in the tool itself (dependencies are
only ever installed into a *generated app's* own venv, never boyoapps'):

### `common.py` — shared curses helpers (color pairs, crash-safe writes)

```python
"""Shared curses helpers used by boyoapps' selector, editor, and the
create/edit/delete flows -- centralized so all three agree on color pairs
and none of them can crash on a write that touches the terminal's
bottom-right cell (a real ncurses gotcha, not theoretical -- it crashed
an earlier app in this family until every write got wrapped)."""
import curses

PAIR_SELECTED = 1
PAIR_TITLE = 2
PAIR_KEYWORD = 3
PAIR_STRING = 4
PAIR_COMMENT = 5
PAIR_NUMBER = 6
PAIR_WARN = 7


def init_colors():
    try:
        curses.start_color()
        curses.use_default_colors()
        curses.init_pair(PAIR_SELECTED, curses.COLOR_BLACK, curses.COLOR_CYAN)
        curses.init_pair(PAIR_TITLE, curses.COLOR_CYAN, -1)
        curses.init_pair(PAIR_KEYWORD, curses.COLOR_YELLOW, -1)
        curses.init_pair(PAIR_STRING, curses.COLOR_GREEN, -1)
        curses.init_pair(PAIR_COMMENT, curses.COLOR_BLUE, -1)
        curses.init_pair(PAIR_NUMBER, curses.COLOR_MAGENTA, -1)
        curses.init_pair(PAIR_WARN, curses.COLOR_RED, -1)
        return True
    except curses.error:
        return False


def safe_addnstr(stdscr, y, x, text, n, *attr):
    try:
        stdscr.addnstr(y, x, text, n, *attr)
    except curses.error:
        pass


def safe_chgat(stdscr, y, x, n, attr):
    try:
        stdscr.chgat(y, x, n, attr)
    except curses.error:
        pass


def read_line(stdscr, y, prompt, initial=""):
    """Blocking single-line text entry at row y. Enter accepts, Esc
    cancels (returns None). Backspace only -- no arrow-key editing, so
    it stays usable from any mobile ssh client."""
    curses.curs_set(1)
    stdscr.timeout(-1)
    text = initial
    while True:
        height, width = stdscr.getmaxyx()
        safe_addnstr(stdscr, y, 0, " " * width, width)
        safe_addnstr(stdscr, y, 0, (prompt + text)[:width], width)
        stdscr.move(min(y, height - 1), min(width - 1, len(prompt) + len(text)))
        stdscr.refresh()
        key = stdscr.getch()
        if key in (curses.KEY_ENTER, 10, 13):
            curses.curs_set(0)
            return text
        if key == 27:
            curses.curs_set(0)
            return None
        if key in (curses.KEY_BACKSPACE, 127, 8):
            text = text[:-1]
        elif key == curses.KEY_RESIZE:
            continue
        elif 32 <= key < 127:
            text += chr(key)


def confirm(stdscr, y, prompt, options="y"):
    """Blocking single-keypress confirm. `options` lists the accepting
    chars (lowercase); n/q/Esc always cancel (return None) regardless of
    `options`, so callers never have to special-case them."""
    stdscr.timeout(-1)
    width = stdscr.getmaxyx()[1]
    safe_addnstr(stdscr, y, 0, " " * width, width)
    safe_addnstr(stdscr, y, 0, prompt, width)
    stdscr.refresh()
    while True:
        key = stdscr.getch()
        if key == curses.KEY_RESIZE:
            continue
        if key in (27, ord("n"), ord("q")):
            return None
        ch = chr(key).lower() if 0 <= key < 256 else ""
        if ch in options:
            return ch
```

### `editor.py` — the in-menu Python code editor

```python
"""Minimal curses editor for the Python scripts boyoapps generates:
cursor movement, auto-indent on Enter, and tokenize-based syntax
highlighting. Not a general text editor -- built for editing one small
script at a time, from within boyoapps' create/edit flow."""
import curses
import io
import keyword
import sys
import tokenize

try:
    import termios
except ImportError:  # not on a real tty (piped/non-posix) -- flow control is moot
    termios = None

from common import (
    PAIR_COMMENT, PAIR_KEYWORD, PAIR_NUMBER, PAIR_STRING,
    confirm, safe_addnstr, safe_chgat,
)

INDENT = "    "


def _disable_flow_control():
    """^S/^Q are XON/XOFF flow-control chars at the tty layer -- without
    this, the terminal driver eats them to pause/resume output and they
    never reach getch() at all (silently, no error -- just does nothing).
    Same fix nano and other terminal editors apply. Returns the prior
    termios state so it can be restored on exit."""
    if termios is None or not sys.stdin.isatty():
        return None
    try:
        fd = sys.stdin.fileno()
        old = termios.tcgetattr(fd)
        new = list(old)
        new[0] &= ~(termios.IXON | termios.IXOFF)
        termios.tcsetattr(fd, termios.TCSANOW, new)
        return old
    except termios.error:
        return None


def _restore_flow_control(old):
    if old is None or termios is None:
        return
    try:
        termios.tcsetattr(sys.stdin.fileno(), termios.TCSANOW, old)
    except termios.error:
        pass


def load_lines(path):
    if path.exists():
        text = path.read_text()
        lines = text.split("\n")
        if lines and lines[-1] == "":
            lines.pop()
        return lines or [""]
    return [""]


def save_lines(path, lines):
    path.write_text("\n".join(lines) + "\n")


def _classify(tok_type, tok_str):
    if tok_type == tokenize.COMMENT:
        return PAIR_COMMENT
    if tok_type == tokenize.STRING:
        return PAIR_STRING
    if tok_type == tokenize.NUMBER:
        return PAIR_NUMBER
    if tok_type == tokenize.NAME and (keyword.iskeyword(tok_str) or keyword.issoftkeyword(tok_str)):
        return PAIR_KEYWORD
    return None


def highlight(lines):
    """{line_idx: [(col_start, col_end, pair), ...]}, or None if the
    buffer doesn't currently tokenize cleanly (e.g. an unterminated
    string mid-edit) -- caller falls back to plain rendering for that
    frame and highlighting resumes once the syntax is valid again."""
    src = "\n".join(lines) + "\n"
    segs = {}
    try:
        for tok in tokenize.generate_tokens(io.StringIO(src).readline):
            pair = _classify(tok.type, tok.string)
            if pair is None:
                continue
            if tok.start[0] != tok.end[0]:
                for row in range(tok.start[0], tok.end[0] + 1):
                    li = row - 1
                    if li < 0 or li >= len(lines):
                        continue
                    c0 = tok.start[1] if row == tok.start[0] else 0
                    c1 = tok.end[1] if row == tok.end[0] else len(lines[li])
                    segs.setdefault(li, []).append((c0, c1, pair))
            else:
                li = tok.start[0] - 1
                if 0 <= li < len(lines):
                    segs.setdefault(li, []).append((tok.start[1], tok.end[1], pair))
    except (tokenize.TokenError, IndentationError, SyntaxError, ValueError, IndexError):
        return None
    return segs


def _auto_indent(prefix):
    stripped = prefix.rstrip()
    indent = prefix[:len(prefix) - len(prefix.lstrip(" "))]
    if stripped.endswith(":"):
        indent += INDENT
    return indent


def run_editor(stdscr, path):
    """Returns (saved_any, lines) -- saved_any tells the caller whether
    to re-run dependency scanning on the way out."""
    old_termios = _disable_flow_control()
    try:
        return _run_editor_loop(stdscr, path)
    finally:
        _restore_flow_control(old_termios)


def _run_editor_loop(stdscr, path):
    lines = load_lines(path)
    cy, cx = 0, 0
    top, xoff = 0, 0
    modified = False
    saved_any = False
    status = ""

    curses.curs_set(1)
    stdscr.timeout(-1)

    while True:
        height, width = stdscr.getmaxyx()
        body_h = max(1, height - 2)
        gutter = (len(str(len(lines))) + 1) if width >= 50 else 0
        text_w = max(10, width - gutter)

        if cy < top:
            top = cy
        if cy >= top + body_h:
            top = cy - body_h + 1
        if cx < xoff:
            xoff = cx
        if cx >= xoff + text_w:
            xoff = cx - text_w + 1

        stdscr.erase()
        title = f" {path.name}{' *' if modified else ''} "
        safe_addnstr(stdscr, 0, 0, title.ljust(width), width, curses.A_REVERSE)

        segs = highlight(lines)
        for row in range(body_h):
            li = top + row
            if li >= len(lines):
                continue
            y = row + 1
            line = lines[li]
            if gutter:
                safe_addnstr(stdscr, y, 0, str(li + 1).rjust(gutter - 1), gutter - 1, curses.A_DIM)
            visible = line[xoff:xoff + text_w]
            safe_addnstr(stdscr, y, gutter, visible, text_w)
            if segs is not None:
                for c0, c1, pair in segs.get(li, []):
                    vs, ve = max(c0, xoff), min(c1, xoff + text_w)
                    if ve <= vs:
                        continue
                    safe_chgat(stdscr, y, gutter + vs - xoff, ve - vs, curses.color_pair(pair))

        help_line = f"^S save  ^Q quit  Tab indent  ln {cy + 1}/{len(lines)}"
        if status:
            help_line += f"  -- {status}"
        safe_addnstr(stdscr, height - 1, 0, help_line.ljust(width), width, curses.A_REVERSE)

        cursor_y = min(height - 2, cy - top + 1)
        cursor_x = min(width - 1, gutter + cx - xoff)
        stdscr.move(max(1, cursor_y), max(0, cursor_x))
        stdscr.refresh()

        key = stdscr.getch()
        status = ""

        if key == curses.KEY_RESIZE:
            continue
        elif key == 19:  # ^S
            save_lines(path, lines)
            modified = False
            saved_any = True
            status = "saved"
        elif key in (17, 27):  # ^Q or Esc
            if modified:
                ans = confirm(stdscr, height - 1, "discard unsaved changes? [y] discard  [n] cancel > ", "y")
                if ans != "y":
                    continue
            curses.curs_set(0)
            return saved_any, lines
        elif key == curses.KEY_UP:
            if cy > 0:
                cy -= 1
                cx = min(cx, len(lines[cy]))
        elif key == curses.KEY_DOWN:
            if cy < len(lines) - 1:
                cy += 1
                cx = min(cx, len(lines[cy]))
        elif key == curses.KEY_LEFT:
            if cx > 0:
                cx -= 1
            elif cy > 0:
                cy -= 1
                cx = len(lines[cy])
        elif key == curses.KEY_RIGHT:
            if cx < len(lines[cy]):
                cx += 1
            elif cy < len(lines) - 1:
                cy += 1
                cx = 0
        elif key == curses.KEY_HOME:
            cx = 0
        elif key == curses.KEY_END:
            cx = len(lines[cy])
        elif key == curses.KEY_PPAGE:
            cy = max(0, cy - body_h)
            cx = min(cx, len(lines[cy]))
        elif key == curses.KEY_NPAGE:
            cy = min(len(lines) - 1, cy + body_h)
            cx = min(cx, len(lines[cy]))
        elif key in (curses.KEY_ENTER, 10, 13):
            indent = _auto_indent(lines[cy][:cx])
            rest = lines[cy][cx:]
            lines[cy] = lines[cy][:cx]
            lines.insert(cy + 1, indent + rest)
            cy += 1
            cx = len(indent)
            modified = True
        elif key in (curses.KEY_BACKSPACE, 127, 8):
            if cx > 0:
                lines[cy] = lines[cy][:cx - 1] + lines[cy][cx:]
                cx -= 1
                modified = True
            elif cy > 0:
                prev_len = len(lines[cy - 1])
                lines[cy - 1] += lines[cy]
                del lines[cy]
                cy -= 1
                cx = prev_len
                modified = True
        elif key == curses.KEY_DC:
            if cx < len(lines[cy]):
                lines[cy] = lines[cy][:cx] + lines[cy][cx + 1:]
                modified = True
            elif cy < len(lines) - 1:
                lines[cy] += lines[cy + 1]
                del lines[cy + 1]
                modified = True
        elif key == ord("\t"):
            lines[cy] = lines[cy][:cx] + INDENT + lines[cy][cx:]
            cx += len(INDENT)
            modified = True
        elif 32 <= key < 127:
            ch = chr(key)
            lines[cy] = lines[cy][:cx] + ch + lines[cy][cx:]
            cx += 1
            modified = True
```

### `appgen.py` — scaffolding, registry, and per-app venv reconciliation

```python
"""Registry management plus the create/edit/delete mechanics for apps
made through boyoapps itself: scaffolding a new app, tracking it in
apps.json with managed=True (the marker that gates the in-menu editor --
hand-built apps like tix/portfolio/printer never get that flag), and
per-app dependency installation into <app>/.venv on save so a generated
app's third-party imports are already satisfied the first time it runs."""
import ast
import json
import re
import shutil
import subprocess
import sys
from datetime import date
from pathlib import Path

TERMINAL_APPS_DIR = Path(__file__).resolve().parent.parent
REGISTRY = TERMINAL_APPS_DIR / "apps.json"
BIN_DIR = Path.home() / "bin"

NAME_RE = re.compile(r"^[a-z][a-z0-9_-]{1,30}$")

STDLIB = getattr(sys, "stdlib_module_names", frozenset())

# Import name -> pip package name, for the common cases where they differ.
PACKAGE_OVERRIDES = {
    "cv2": "opencv-python",
    "yaml": "pyyaml",
    "PIL": "pillow",
    "bs4": "beautifulsoup4",
    "dotenv": "python-dotenv",
    "serial": "pyserial",
    "Crypto": "pycryptodome",
    "dateutil": "python-dateutil",
    "sklearn": "scikit-learn",
    "docx": "python-docx",
    "usb": "pyusb",
    "attr": "attrs",
    "requests_oauthlib": "requests-oauthlib",
}

TEMPLATE = '''#!/usr/bin/env python3
"""{description}"""


def main():
    pass


if __name__ == "__main__":
    main()
'''


def load_apps():
    with open(REGISTRY) as f:
        return json.load(f)


def save_apps(apps):
    with open(REGISTRY, "w") as f:
        json.dump(apps, f, indent=2)
        f.write("\n")


def validate_command_name(name):
    if not NAME_RE.match(name):
        return False, "lowercase letters/digits/-/_ only, must start with a letter, 2-31 chars"
    if any(a["command"] == name for a in load_apps()):
        return False, f"'{name}' is already a registered app"
    if shutil.which(name):
        return False, f"'{name}' collides with an existing system command"
    return True, ""


def parse_third_party_imports(source):
    try:
        tree = ast.parse(source)
    except SyntaxError:
        return []
    mods = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                mods.add(alias.name.split(".")[0])
        elif isinstance(node, ast.ImportFrom):
            if node.level == 0 and node.module:
                mods.add(node.module.split(".")[0])
    return sorted(m for m in mods if m and m.isidentifier() and m not in STDLIB and m != "__future__")


def set_shebang(main_py, interpreter):
    lines = main_py.read_text().splitlines(keepends=True)
    new_first = f"#!{interpreter}\n"
    if lines and lines[0].startswith("#!"):
        lines[0] = new_first
    else:
        lines.insert(0, new_first)
    main_py.write_text("".join(lines))


def venv_python(app_dir):
    return app_dir / ".venv" / "bin" / "python"


def ensure_dependencies(app_dir, source, log=lambda msg: None):
    """Ensures every third-party top-level import in `source` is
    installed into app_dir/.venv (created on first need), and points
    main.py's shebang at that interpreter -- or back at plain python3 if
    there's nothing third-party to isolate. Returns (ok, failed_pkgs)."""
    main_py = app_dir / "main.py"
    mods = parse_third_party_imports(source)
    if not mods:
        set_shebang(main_py, "/usr/bin/env python3")
        return True, []

    vpy = venv_python(app_dir)
    if not vpy.exists():
        log(f"creating venv for {app_dir.name}...")
        r = subprocess.run(
            [sys.executable, "-m", "venv", str(app_dir / ".venv")],
            capture_output=True, text=True,
        )
        if r.returncode != 0:
            log(f"venv creation failed: {r.stderr.strip()[:200]}")
            return False, mods

    failed = []
    for mod in mods:
        check = subprocess.run([str(vpy), "-c", f"import {mod}"], capture_output=True)
        if check.returncode == 0:
            continue
        pkg = PACKAGE_OVERRIDES.get(mod, mod)
        log(f"installing {pkg}...")
        r = subprocess.run(
            [str(vpy), "-m", "pip", "install", "--quiet", pkg],
            capture_output=True, text=True,
        )
        if r.returncode != 0:
            log(f"failed to install {pkg}: {r.stderr.strip()[:200]}")
            failed.append(pkg)

    set_shebang(main_py, str(vpy))
    return (len(failed) == 0), failed


def create_app(command, description, log=lambda msg: None):
    ok, err = validate_command_name(command)
    if not ok:
        return False, err

    app_dir = TERMINAL_APPS_DIR / command
    if app_dir.exists():
        return False, f"{app_dir} already exists"

    app_dir.mkdir(parents=True)
    main_py = app_dir / "main.py"
    main_py.write_text(TEMPLATE.format(description=description or command))
    main_py.chmod(0o755)

    bin_link = BIN_DIR / command
    if bin_link.exists() or bin_link.is_symlink():
        bin_link.unlink()
    bin_link.symlink_to(main_py)

    apps = load_apps()
    apps.append({
        "name": command,
        "command": command,
        "description": description or command,
        "path": str(app_dir),
        "added": date.today().isoformat(),
        "managed": True,
    })
    save_apps(apps)
    return True, ""


def delete_app(entry):
    if entry["command"] == "boyoapps":
        return False, "can't delete boyoapps from within itself"

    app_dir = Path(entry["path"]).resolve()
    if app_dir.parent != TERMINAL_APPS_DIR.resolve():
        return False, f"refusing to delete outside {TERMINAL_APPS_DIR}: {app_dir}"

    bin_link = BIN_DIR / entry["command"]
    try:
        if bin_link.is_symlink() or bin_link.exists():
            bin_link.unlink()
    except OSError:
        pass

    if app_dir.exists():
        shutil.rmtree(app_dir)

    apps = [a for a in load_apps() if a["command"] != entry["command"]]
    save_apps(apps)
    return True, ""
```

### `main.py` — the selector menu and entry point

```python
#!/usr/bin/env python3
import curses
import os
import shutil
import subprocess
import sys
import textwrap
import time
from pathlib import Path

from appgen import create_app, delete_app, ensure_dependencies, load_apps
from common import confirm, init_colors, read_line, safe_addnstr
from editor import run_editor

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
    """Same figlet+lolcat house style as the ssh shell rice, but width-
    checked first -- measures the actual rendered width per font against
    the real terminal and picks the widest one that fits, so it doesn't
    wrap mid-glyph on a narrow mobile terminal the way `clear`'s banner
    can. Falls back to plain colored text if even the smallest font
    doesn't fit."""
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
        print(f"\033[1m\033[36m{name}\033[0m")
        time.sleep(0.2)
        return

    try:
        lol = subprocess.run(["lolcat", "-f"], input=art, capture_output=True, text=True, timeout=2)
        print(lol.stdout if lol.returncode == 0 else art, end="")
    except (FileNotFoundError, subprocess.TimeoutExpired):
        print(art, end="")
    time.sleep(0.3)


def print_plain(apps):
    width = max(len(a["command"]) for a in apps) + 2
    for app in apps:
        print(f"{app['command']:<{width}}{app['description']}")
    print(f"\n{len(apps)} app(s) registered. Run any by typing its command name.")


def _new_app_flow(stdscr, height, log):
    name = read_line(stdscr, height - 1, "new app command name: ")
    if not name:
        return None, ""
    name = name.strip()
    desc = read_line(stdscr, height - 1, "one-line description: ")
    if desc is None:
        return None, ""
    ok, err = create_app(name, desc.strip(), log)
    if not ok:
        return None, f"create failed: {err}"

    apps = load_apps()
    entry = next(a for a in apps if a["command"] == name)
    path = Path(entry["path"]) / "main.py"
    saved, new_lines = run_editor(stdscr, path)
    if saved:
        ensure_dependencies(Path(entry["path"]), "\n".join(new_lines), log)
    return name, f"created '{name}'"


def _edit_app_flow(stdscr, entry, log):
    path = Path(entry["path"]) / "main.py"
    saved, new_lines = run_editor(stdscr, path)
    if saved:
        ensure_dependencies(Path(entry["path"]), "\n".join(new_lines), log)
        return "saved"
    return ""


def _delete_app_flow(stdscr, height, entry):
    prompt = (
        f"delete '{entry['command']}'?  [y] yes  [n] cancel  "
        f"(dir+symlink+registry, permanent -- {entry['path']})"
    )
    ans = confirm(stdscr, height - 1, prompt, "y")
    if ans != "y":
        return ""
    ok, err = delete_app(entry)
    if not ok:
        return f"delete failed: {err}"
    return f"deleted '{entry['command']}'"


def run_selector(stdscr, apps):
    curses.curs_set(0)
    has_color = init_colors()
    if has_color:
        selected_attr = curses.color_pair(1)
        title_attr = curses.color_pair(2) | curses.A_BOLD
    else:
        selected_attr = curses.A_REVERSE
        title_attr = curses.A_BOLD

    idx = 0
    status = ""
    while True:
        apps = load_apps()
        idx = min(idx, max(0, len(apps) - 1))
        if not apps:
            return None

        stdscr.erase()
        height, width = stdscr.getmaxyx()

        title = "boyo's terminal apps"
        stdscr.attron(title_attr)
        safe_addnstr(stdscr, 0, max(0, (width - len(title)) // 2), title, width)
        stdscr.attroff(title_attr)

        footer_h = 5
        list_top = 2
        list_bottom = max(list_top, height - footer_h)
        visible = max(1, list_bottom - list_top)

        offset = 0
        if idx >= visible:
            offset = idx - visible + 1

        for row, app in enumerate(apps[offset:offset + visible]):
            actual_i = row + offset
            y = list_top + row
            if y >= list_bottom:
                break
            marker = "> " if actual_i == idx else "  "
            tag = " [managed]" if app.get("managed") else ""
            line = f"{marker}{app['command']}{tag}"
            if actual_i == idx:
                stdscr.attron(selected_attr)
                safe_addnstr(stdscr, y, 0, line.ljust(width), width)
                stdscr.attroff(selected_attr)
            else:
                safe_addnstr(stdscr, y, 0, line, width)

        sel = apps[idx]
        desc_lines = textwrap.wrap(sel["description"], max(10, width - 1)) or [""]
        fy = list_bottom + 1
        for i, dl in enumerate(desc_lines[:2]):
            if fy + i < height - 1:
                safe_addnstr(stdscr, fy + i, 0, dl, width)

        short_path = sel["path"].replace(str(Path.home()), "~", 1)
        meta = f"{short_path}  (added {sel['added']})"
        safe_addnstr(stdscr, min(fy + 2, height - 2), 0, meta, width)

        help_text = "j/k move  enter run  n new  e edit  d delete  q quit"
        if status:
            help_text += f"  -- {status}"
        safe_addnstr(stdscr, height - 1, 0, help_text[:width].ljust(width), width)

        stdscr.refresh()

        key = stdscr.getch()
        status = ""

        def log(msg):
            safe_addnstr(stdscr, height - 1, 0, msg.ljust(width), width)
            stdscr.refresh()

        if key in (curses.KEY_UP, ord('k')):
            idx = (idx - 1) % len(apps)
        elif key in (curses.KEY_DOWN, ord('j')):
            idx = (idx + 1) % len(apps)
        elif key in (curses.KEY_ENTER, 10, 13):
            return apps[idx]
        elif key in (ord('q'), 27):
            return None
        elif key == curses.KEY_RESIZE:
            continue
        elif key == ord('n'):
            new_name, status = _new_app_flow(stdscr, height, log)
            curses.curs_set(0)
            if new_name:
                fresh = load_apps()
                idx = next((i for i, a in enumerate(fresh) if a["command"] == new_name), idx)
        elif key == ord('e'):
            if not apps[idx].get("managed"):
                status = "edit is only available for boyoapps-created apps"
            else:
                status = _edit_app_flow(stdscr, apps[idx], log)
                curses.curs_set(0)
        elif key == ord('d'):
            status = _delete_app_flow(stdscr, height, apps[idx])


def main():
    apps = load_apps()
    if not apps:
        print("No apps registered yet.")
        return

    if not sys.stdout.isatty() or "--list" in sys.argv:
        print_plain(apps)
        return

    print_banner("boyoapps")
    chosen = curses.wrapper(run_selector, apps)
    if chosen is None:
        return

    command = chosen["command"]
    try:
        os.execvp(command, [command])
    except OSError as e:
        print(f"boyoapps: couldn't run '{command}': {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

## Master Prompt

Paste this to an AI coding assistant (e.g. Claude) to have it build the same
kind of tool for you:

```
Build me a bare-word terminal app launcher and Python scaffolding tool,
designed first for usability while remoted in over SSH -- including from a
phone SSH client on a narrow, laggy terminal, not just a full desktop
terminal emulator.

1. A registry file (JSON) listing small personal scripts by command name,
   description, filesystem path, and date added. A launcher menu (curses-
   based) lists them, lets me move with arrow keys or j/k, and running one
   should genuinely replace the launcher process (exec, not spawn-and-wait)
   so it behaves like I typed the command directly.

2. In-menu scaffolding: a keypress prompts for a new command name and a
   one-line description, validates the name (lowercase, no collision with
   an existing registered app or an existing system command), generates a
   minimal main.py from a template, symlinks it into a directory already on
   PATH, registers it, and drops straight into a built-in code editor to
   write it.

3. The built-in code editor should be a real, if minimal, curses text
   editor: cursor movement, backspace/delete, auto-indent that continues
   the previous line's indentation and adds a level after a line ending in
   ":", a persistent status/help line, and syntax highlighting using only
   the language's own standard-library tokenizer -- not a third-party
   syntax highlighting package, since a freshly scaffolded script has no
   dependencies installed yet and the editor can't assume any exist. Guard
   every single screen write against the terminal-library's own edge-case
   crash (writing to the bottom-right cell of the screen is a classic
   ncurses/curses gotcha) rather than relying on not hitting it.

4. Explicitly handle the SSH/mobile constraint: if your terminal library's
   save/quit keybindings collide with tty-level flow-control characters
   (^S/^Q are the classic XON/XOFF pause/resume signals and get silently
   eaten by the terminal driver otherwise), disable flow control on the
   underlying file descriptor for the duration of the editor session and
   restore it on exit. Keep single-line text-entry prompts (e.g. "enter a
   name") backspace-only with no arrow-key editing, since that's more
   likely to survive an unusual mobile keyboard's key-code mapping. Make
   any fixed-width UI element (like a line-number gutter) collapse or hide
   below some minimum terminal width instead of cramping a narrow screen.

5. On every save from the editor, re-parse the script's own top-level
   imports (via the language's AST, not a regex) and reconcile a per-app
   virtual environment to match: create it on first need, install anything
   that isn't a standard-library module, and repoint the script's shebang
   at that environment's interpreter -- or back at the plain system
   interpreter if nothing third-party remains. Maintain a small table of
   cases where the import name and the package name differ (e.g. `cv2` ->
   `opencv-python`, `yaml` -> `pyyaml`, `PIL` -> `pillow`) so the common
   ones just work without me having to know the mapping.

6. A delete flow with confirmation that cleans up all three things
   together: the symlink, the script's own directory, and its registry
   entry -- and refuses to delete the launcher tool from within itself.

The whole point is that creating, writing, and running a new small script
should be possible start-to-finish from inside one SSH session, on
whatever terminal happens to be at hand, without ever needing a second
tool or a desktop environment.
```

## Notes

- Built with Claude Code. No external Python packages required to run the
  tool itself — `figlet`/`lolcat` are optional shell-level niceties for the
  startup banner and degrade gracefully if absent.
- Third-party dependencies only ever get installed into a *generated app's*
  own `.venv`, never into the launcher's own environment.
