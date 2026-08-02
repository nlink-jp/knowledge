# Containers & Infra

Podman on macOS, and databases/rendering libraries inside containers. Each entry
follows **symptom → why → how to apply**.

---

## macOS + Podman Machine differs substantially from native Linux containers

**Symptom:** Live testing of a containerized tool hit multiple problems, all
rooted in the VM layer (Apple Hypervisor + virtiofs + gvproxy).

**How to apply (checklist):**
- **Unix domain sockets cannot cross virtiofs** (host agent sockets can't be
  passed into containers).
- gvproxy can retain ports — sometimes ports stay bound after the container
  stops.
- Build-time OOM: the default Podman Machine memory (~4 GB) can be insufficient;
  8 GB recommended.
- Services inside containers must bind **`0.0.0.0`** (`127.0.0.1` is unreachable
  from outside the VM).
- sshd does not pass Dockerfile `ENV` to child processes — put PATH etc. in
  `.profile`/`.bashrc`.
- macOS `sed -i` differs from Linux — use the `sed + mv` pattern.
- macOS `date -j -f` ignores timezone info — take epoch values directly.

## Don't pre-touch DuckDB files; bind-mount the parent directory

**Symptom:** Touching an empty file on the host before bind-mounting makes DuckDB
fail with `IOException: not a valid DuckDB database file`.

**Why:** DuckDB safely errors on 0-byte files — it cannot distinguish "empty"
from "corrupt".

**How to apply:** Place the DB file inside a work directory and mount **only the
parent** (`-v <host>/work:/work`). Let DuckDB create the file on first
connection. Write no pre-touch code.

## matplotlib (Agg) draws everything with the first valid font in font.sans-serif

**Symptom:** With `font.sans-serif: DejaVu Sans, Noto Sans CJK JP, ...`, Japanese
labels became tofu with `Glyph ... missing from font(s) DejaVu Sans` warnings.

**Why:** The Agg backend **locks in the first valid font found** — there is no
per-glyph fallback.

**How to apply:** Put the CJK font first:
`font.sans-serif: Noto Sans CJK JP, DejaVu Sans, ...`.
Noto Sans CJK JP covers Latin/digits/symbols too, so leading with it has no side
effects on English labels. Verify with `python -W error::UserWarning` rendering a
Japanese title without warnings.
