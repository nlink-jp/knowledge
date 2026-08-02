# Shell Scripting

Bash pitfalls. Each entry follows **symptom → why → how to apply**.

---

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
