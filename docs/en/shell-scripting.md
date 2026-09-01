# Shell Scripting

Bash / zsh / sed pitfalls. Each entry follows **symptom → why → how to apply**.

---

## macOS sed is BSD sed — GNU-incompatible, and which one runs is environment-dependent

**Symptom:** Scripts that work on Linux (GNU sed) break on macOS and vice versa —
a recurring class of trouble. Worse, on machines with Homebrew GNU sed (`gsed`)
installed, *which* sed runs becomes environment-dependent, so "it worked on my
machine" says nothing about portability.

**Why (the major incompatibilities):**
- **`-i` (in-place) syntax differs.** BSD sed requires an extension argument
  (`sed -i '' -e 's/a/b/' f`); GNU sed takes bare `-i`. `sed -i '' ...` breaks
  GNU (treated as an empty filename); bare `-i` makes BSD sed eat the next
  argument as the extension.
- GNU extensions (BRE `\+` `\|`, the `s///I` flag, `sed -z`, …) don't exist in
  BSD sed. Line-continuation for `a`/`i`/`c` also differs (BSD needs `a\` +
  newline).
- **Aliases are interactive-only**: `alias sed=gsed` in interactive zsh does not
  affect `sed` inside scripts, which resolve via PATH. Conversely, putting
  Homebrew's gnubin (`$(brew --prefix)/opt/gnu-sed/libexec/gnubin`) on PATH
  switches scripts to GNU sed too, diverging from stock macOS. This structure is
  why one-liners that worked interactively break in scripts, and vice versa.

**How to apply:**
- Safest in scripts: **avoid `-i` entirely** — standardize on
  `sed ... > tmp && mv tmp file` (identical behavior on containers/Linux and
  macOS). If `-i` must be used, restrict to the **attached form `-i.bak`**
  (accepted by both) and delete the `.bak` afterwards.
- When GNU features are needed, don't rely on whatever `sed` is: **call `gsed`
  explicitly**, check with `command -v gsed`, and fail with a clear message when
  missing.
- Runtime detection: `sed --version` answers only for GNU (BSD sed errors).
- Review trigger: on seeing `sed -i`, always ask "on which OS, with which sed?"

## zsh parameter expansion is not bash — no word splitting, no globbing

**Symptom:** Pasting bash-oriented one-liners and snippets into macOS's default
interactive shell (zsh) changes their behavior — another recurring class of
trouble.

**Why (the major differences):**
- **Unquoted `$var` is not word-split** (`SH_WORD_SPLIT` off by default):
  `opts="-l -a"; ls $opts` splits into two args in bash but passes the single
  word `"-l -a"` in zsh.
- **Unquoted `$var` is not glob-expanded** (`GLOB_SUBST` off by default):
  `pat='*.md'; ls $pat` passes the literal `*.md` in zsh.
- **Unmatched globs are errors** (`NOMATCH` on by default): bash passes the
  pattern through literally; zsh aborts with `zsh: no matches found` **before
  running the command**. `scp host:*.gz .` failing on the local glob is the
  classic case.
- **Arrays are 1-indexed**, and `$array` expands to all elements (bash: first
  element only).

**How to apply:**
- **Give scripts a `#!/bin/bash` (or `#!/usr/bin/env bash`) shebang and run them
  as bash**, independent of the interactive shell's dialect. Avoid `sh file.sh`
  and `source`-ing (both change the interpreter).
- When targeting zsh, use the explicit operators: `${=var}` for word splitting,
  `${~var}` for globbing, `*.tmp(N)` (or `setopt null_glob`) to tolerate no
  matches. Hold argument lists as **arrays**, not strings
  (`opts=(-l -a); ls $opts`).
- Always quote globs meant to expand remotely (scp/ssh).
- One-liners in documentation either state "run under bash" or are written to
  behave identically in both shells.

## trap handlers run outside function scope

**Symptom:** A cleanup trap handler referenced `local` variables and died with
unbound-variable errors under `set -u` (the same trap was hit three times).

**How to apply:**
- Declare every variable a trap handler touches as global.
- Under `set -e`, a non-zero command before the trap kills the script — protect
  with `|| true`.
- Trap `INT TERM`, not just `EXIT`.

## Never inline `$(...)` in the pattern of `${var//pattern/replacement}`

**Symptom:** A substitution loop resolving `$(VAR)` references **silently** did
nothing. No error, so the cause stayed invisible until a test was written.

```bash
s='$(OUT)'; val='dist/tool'
echo "${s//\$(OUT)/$val}"               # => $(OUT)     not replaced (no error either)
pat='$(OUT)'; echo "${s//"$pat"/$val}"  # => dist/tool  correct
```

**How to apply:** For placeholder substitution in bash, put the pattern in a
variable and write `${s//"$pat"/$rep}`. Suspect any existing inline form. Never
"simplify" it back inline (leave a prohibition comment in the code).

## An operational script must not infer its root from $PWD — and a skipped check must never summarize green

**Symptom:** an org-wide health-check script defaulted its target root to
`$(pwd)`. Run by an agent whose shell had silently persisted in another
repository's directory, every repo came back "not found locally" — as a
warning — and the run still printed **"Result: all checks passed." with
exit 0, having checked nothing**. The wrong-cwd invocation was not a
one-off: an agent environment's working directory persists invisibly
between shell calls and drifts away from what the operator believes.

**Why:** two defects compounding. (1) The script could derive its root
from its own location (`BASH_SOURCE`) but delegated it to the caller's
cwd — the least trustworthy input deciding the most important question.
(2) Missing repositories were warnings, not counted failures — a
fail-open skip that made unexamined targets look like passing ones.
Either alone is survivable (with only the first, an INCOMPLETE verdict
surfaces the mistake; with only the second, the right root is used);
composed, they manufacture a green result out of zero checks.

**How to apply:**
- A script that lives inside the tree it examines derives its root
  **relative to `BASH_SOURCE`** (`SCRIPT_DIR/../..`), keeping an explicit
  argument as the override. A `$PWD` default is an unverified assumption
  that the caller stands in the right place.
- A checking script **separates skips from passes**: targets it could not
  find or read are counted, the summary says "INCOMPLETE — N not
  checked", and the exit status is non-zero. Green means "every target
  examined, every check passed" and nothing less — a check must never
  claim more than it verified.
- The operator-side twin habit: prefix agent shell calls with
  `cd /absolute/path &&`. But discipline slips somewhere eventually,
  which is why the script-side mechanism is the real wall.
