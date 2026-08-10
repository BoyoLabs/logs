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
