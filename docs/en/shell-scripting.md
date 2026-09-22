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

## `|| true` at the end of an `&&` chain forgives the whole chain, not the last command

**Symptom:** A release check was written like this:

```sh
unzip -q "$zip" -d "$tmp" && "$tmp/$bin" --version && spctl -a "$tmp/$bin" | head -2 || true
```

The `|| true` was written for the informational `spctl` probe. But a zip that did not unpack, or
a binary that did not run, still ended with the check printing OK.

**Why:** `A && B && C || true` parses as `(A && B && C) || true`. `||` takes the whole preceding
list as its left operand, not the command immediately before it — so a `|| true` written to
excuse one command swallows every mandatory gate in front of it.

**How to apply:** Put a command whose failure is acceptable in its **own statement**. Gate each
mandatory step separately with its own message: `|| { echo …; exit 1; }`. A pipeline ending in
`head` has the same shape — the exit status is `head`'s — and is not a reason to add `|| true`.

## `cmd | grep -q` under `set -o pipefail` fails on a match

**Symptom:** A release-verification script checked the signature of a downloaded binary:

```sh
set -o pipefail
codesign -dvv "$bin" 2>&1 | grep -q '^Authority=Developer ID Application' || { echo "FAIL: not Developer ID signed"; rc=1; }
```

The line before it — the same command without `-q` — had just printed
`Authority=Developer ID Application: …`. The gate still reported FAIL, twice, on an asset that
was correctly signed.

**Why:** `grep -q` exits at the first match. If the producer still has output to write, its next
write hits a closed pipe and it dies with SIGPIPE (status 141). Under `pipefail` the pipeline's
status is the rightmost non-zero one, so a *successful* match becomes a failure whenever the
producer's output is longer than the matched line. Without `-q`, grep reads to the end and the
producer finishes normally, which is why the neighbouring line looked fine. The behaviour is
racy: a short output may fit in the pipe buffer and pass.

**How to apply:** Capture the producer's output once into a variable and grep that —
`info=$(codesign -dvv "$bin" 2>&1); printf '%s\n' "$info" | grep -q …` — or use a form that
reads everything (`grep -c … >/dev/null`). The usual random-token idiom is the same trap:
`LC_ALL=C tr -dc 'a-z0-9' </dev/urandom | head -c 24` ends with `head` closing the pipe and
`tr` dying with 141, and under `set -euo pipefail` the script aborts at that line (spice-client's
live peer gate, 2026-09-18, before its first `podman run`, with this very section already in
the knowledge base). `openssl rand -hex 16` reads exactly what it needs. Never put `pipefail`, a chatty producer and an
early-exiting consumer (`grep -q`, `head`, `read`) on one pipeline whose status is a gate. When
a gate reports a failure that the same evidence, printed a line earlier, contradicts, suspect
the plumbing before the asset — then fix the check and re-run it for a machine verdict instead
of reasoning the FAIL away.

## On macOS's bash 3.2, an empty array under `set -u` is an unbound variable

**Symptom:** A fixture script built its optional arguments as an array and
expanded them in the middle of a long command:

```sh
set -euo pipefail
AUDIO_ARGS=()
[ -n "${WANT_AUDIO:-}" ] && AUDIO_ARGS=(-audiodev spice,id=a -device virtio-sound-pci,audiodev=a)
podman run ... "${AUDIO_ARGS[@]}" ...
```

With the option off, the script died before doing anything:
`AUDIO_ARGS[@]: unbound variable`.

**Why:** macOS still ships bash 3.2 as `/bin/bash`, and there `"${arr[@]}"` on an
*empty* array counts as unset under `set -u`. Bash 4.4 and later special-case it.
A script with `#!/bin/bash` gets 3.2 on macOS no matter which bash Homebrew
installed, so this is invisible when it is developed or tested under bash 5.

**How to apply:** Expand optional arrays as `${arr[@]+"${arr[@]}"}`, which yields
nothing when the array is empty and the elements otherwise. The same applies to
`$@` in a function with no arguments. Do not reach for `set +u` around the call:
that disables the check for everything else on the line too. And remember the
neighbouring trap in the same construct — a `#` comment cannot be placed inside a
backslash-continued command; it swallows the continuation and the command runs
with the wrong arguments or not at all.

## `$?` after an `if` statement is the statement's own status, not the condition's

**Symptom:** A helper classified failures and retried one of them, passing every other
failure back with its original status:

```sh
retry_on_port_collision() {
    while true; do
        if error="$("$@" 2>&1)"; then return 0; fi
        status=$?                       # wrong
        case "$error" in
            *"address already in use"*) sleep 1 ;;
            *) echo "$error" >&2; return "$status" ;;
        esac
    done
}
```

The unit test for the pass-through path failed: the command exited 125 and the function
returned 0. `error` held the right stderr and `case` took the right branch. Only the
status was lost.

**Why:** `if` is a compound command with a status of its own. When the condition is
false and there is no `else` or `elif`, the `if` statement completes with **0**. Reading
`$?` after `fi` reads that, not the condition. This is POSIX behaviour, identical across
shells. It fails in the forgiving direction, so it shows up as the error path silently
reporting success — the hardest shape to notice.

**How to apply:** Read the condition's status **inside** a branch. Immediately after
entering `else`, `$?` is the status of the condition's last command:

```sh
if error="$("$@" 2>&1)"; then
    return 0
else
    status=$?
fi
```

Writing `cmd && return 0` and reading `$?` next does not work under `set -e`: a failing
left operand makes the whole `&&` list fail and the shell exits there. Condition context
is exempt from `set -e`, which makes `if`/`else` the only form that is safe to write.
The same trap sits after `while`, `until` and `case`. Whenever you want to keep a
condition's status, keep it inside the branch.

## `! grep -q` reads a grep error as "no match" — pass a pattern that starts with `-` through `-e`

**Symptom:** A test checked that no trace of the old form was left in a converted file:
`! grep -qF '--version && \' file`. The test also passed when the conversion rewrote nothing.

**Why:** The pattern starts with `-`, so grep reads it as an **option**. It rejects the unknown
option, prints its usage and exits 2. grep's status has three values (0 match, 1 no match,
2 error). `!` folds them into two and turns 2 into true, as if nothing had matched. A negated
check falls toward passing whenever the check itself did not run.

**How to apply:**
- Always pass the pattern with `-e` (`grep -qF -e "$pat" file`), both for patterns that start
  with `-` and for patterns that come from a variable.
- To check that something is absent, don't stop at `!`. Require 1 explicitly:
  `rc=0; grep -qF -e "$pat" file || rc=$?; [ "$rc" -eq 1 ]`. A 2 then fails, and the form is
  safe under `set -e`.
- Break a negated check on purpose once and watch it fail. Here the only sign was grep's usage
  text mixed into the test output. Anyone reading only the pass/fail count saw a pass.
