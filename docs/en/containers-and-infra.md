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
## rsyslog's logrotate config is a separate package — without it, only some logs grow unbounded

**Symptom:** On a server up for 2.7 years, `/var/log/messages` had reached 62 MB
and 606,000 lines and had never been rotated. `logrotate` itself completed
successfully every day, and logs such as `dnf`, `btmp` and `wtmp` were rotating
correctly.

**Why:** Recent rsyslog ships its logrotate configuration in a separate
**`rsyslog-logrotate` subpackage**. Without it there is no
`/etc/logrotate.d/rsyslog`, and only the five files rsyslog writes — `messages`,
`cron`, `secure`, `maillog`, `spooler` — fall outside rotation. Everything else
keeps rotating, so nothing looks broken.

**How to apply:**
- Check with `ls /etc/logrotate.d/rsyslog`; install `rsyslog-logrotate` if absent.
- To spot the symptom, run `ls -la /var/log/` and look for **a large log with no
  rotated siblings** (`.1` or `-YYYYMMDD`).
- An oversized log contaminates daily log-summary reports (see "Establish a log's
  timestamp semantics before drawing any conclusion about time" in testing).
- Installing it does not rotate immediately: with a `weekly` schedule the first
  rotation can be up to a week away. Force it if the switchover is needed now. If
  the long history is archived before the switch, keep the archive **outside**
  the summary tool's scan path (outside `/var/log`).

## Passing an individual config file to `logrotate -f` discards the global settings

**Symptom:** A forced rotation via `logrotate -f /etc/logrotate.d/rsyslog` ignored
`rotate 4` — the rotated file was deleted instead of kept.

**Why:** logrotate reads **only the file it is given**. Unless
`/etc/logrotate.conf` is passed, the globals declared there (`weekly`,
`rotate 4`, `create`, `dateext`) never apply, and the built-in defaults take over
(`rotate 0`, i.e. keep nothing).

**How to apply:** Pass `/etc/logrotate.conf` even for manual runs:

```
logrotate -d -f /etc/logrotate.conf     # dry run; -d changes nothing
logrotate -f /etc/logrotate.conf
```

Confirm `rotating pattern: ... (N rotations)` in the dry-run output before the
real run. `(no old logs will be kept)` means `rotate` is not in effect. The
systemd daily run uses `ExecStart=/usr/sbin/logrotate /etc/logrotate.conf`, so the
automated path is correct and only manual invocations hit this.

## An SSH liveness check locks out the checker itself on OpenSSH 9.8+

**Symptom:** While investigating a server whose SSH went unreachable every few
minutes — `ping` and DNS answering 100% of the time, SSH alone dead — storage I/O
hang was the working theory. The cause was **the reachability probe and retry
loop being used to investigate**.

**Why:** `PerSourcePenalties`, added in OpenSSH 9.8 and enabled by default in 9.9,
accumulates penalties per source address. The defaults:

```
crash:90 authfail:5 noauth:1 grace-exceeded:10
refuseconnection:10 max:600 min:15
```

A port check with `nc -z` disconnects without authenticating — `noauth`. A timed
out `ssh` is `grace-exceeded`; a killed one is `crash`. Repeated retries pin the
total at `max`, **blocking the source for up to ten minutes**.

**How to apply:**
- **It looks exactly like a storage I/O hang.** Tell them apart from sshd's logs:
  while a penalty is active no connection is recorded at all (packets never reach
  sshd). A disk-induced stall leaves entries such as
  `Timeout before authentication`.
- If SSH is a monitored service, add the monitoring source to
  `PerSourcePenaltyExemptList`.
- Running a retry loop during an investigation makes **the investigation itself
  produce the symptom**. Back off when connections start failing; penalties
  accumulate, so retrying without a pause only deepens the hole.
