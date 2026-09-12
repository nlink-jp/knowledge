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

## A retention period is not a retention depth — ingestion lag eats into what you keep

**Symptom:** Intending to keep 400 days, a partition expiration of 400 days was configured; only
399 days were actually queryable. On top of that, of the 450 days of history the source backfilled,
the oldest 50 days vanished as they landed. **Neither produced an error or a warning.**

**Why:** The retention window is bounded at both ends. The old end is "today − expiration", which
the lifecycle setting deletes. The new end is "today − ingestion lag", which the source has not
published yet. Expiration only moves the old end, so:

```
days actually retained = expiration − ingestion lag + 1
```

For data that arrives 1-3 days late, such as daily rollups, you are always short by the lag. And
when a bulk historical load (backfill) lands in a destination that already has an expiration, the
**oldest partitions expire as fast as they arrive**. Both the load and the deletion are behaving
correctly, so nothing surfaces as an anomaly.

**How to apply:**

- When the retention depth is a requirement, set the expiration to **"days you want + ingestion
  lag"**. If the lag varies, budget for the worst case.
- **If the oldest surviving day equals "today − expiration" exactly, you are deleting it yourself,
  not hitting the source's limit.** A source-side limit leaves the oldest day on a date unrelated to
  your setting, and it does not advance as days pass. Your own expiration makes it advance by one
  day, every day. That is how you tell them apart.
- **Do not set an expiration until the backfill has finished** (or set it above the maximum the
  source can deliver, then wait). A load can take days, and its intermediate state looks sparse and
  broken. Raising the expiration afterwards does not resurrect what was deleted.
- **A default expiration only applies to objects created after it is set.** The one clean moment is
  right after creating the container and before any content exists. Miss it and you must enumerate
  the existing objects and set each one — anything you miss grows without limit.
- **Do not govern a container holding data with different lags by a single default.** When
  per-event data (lag in minutes) and daily rollups (lag in days) share a container, only the
  latter comes up short.
- An expiration promises "delete anything older than this", not "keep this much". It rolls: one day
  drops off the tail every day, and depth never accumulates. Where long-term retention is an
  obligation, provision a separate tier that is never deleted, alongside the expiring hot tier. The
  hot tier's expiration is a setting about query speed and cost, not about preserving evidence.

Origin: retention design for an audit-log export into a data warehouse (2026-09).

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

## Never format a macOS volume as case-sensitive — Apple's own apps break silently

**What happened:** A Photos library lived on an external NVMe formatted as case-sensitive
APFS. `photolibraryd` pinned a CPU core for hours at a time, sometimes for tens of hours.
For a library of 19k photos the database had bloated to 6 GB, and of its 210k detected-face
records **98% were orphans whose referenced asset did not exist**. Repairing the library,
rebuilding the people data, and finally recreating the library from scratch all failed to
help. **Root cause took two days to find.** Every other device syncing the same data was
working normally.

**Why:** Apple explicitly states that a Photos library must not live on a case-sensitive
volume — files may not be found, which can cause data loss. APFS is case-insensitive by
default, but **case-sensitive is offered as a choice at format time**, which is the trap.

macOS has assumed case-insensitivity since the beginning, and software that does not
support it is common. Adobe Creative Cloud refuses to install on a case-sensitive volume
and Steam refuses to start. **An app that refuses outright is the lucky case** — Apple's
own apps do not refuse, they break quietly. The chain observed here:

```
case-sensitive volume
  → asset reference resolution fails ("unexpectedly has no asset", hundreds per minute)
  → records whose target cannot be resolved accumulate as orphans
  → derived records keep being generated from them, growing into the hundreds of thousands
  → the OS tries to deduplicate; cascading deletes plus extra fetches on save go O(n²)
  → the daemon pins the CPU while committing nothing to disk
```

**How to apply:**
- **Check before putting any Apple app library (Photos, Music, and friends) on external
  storage.**

  ```sh
  diskutil info /Volumes/<name> | grep 'File System Personality'
  # "APFS" = fine / "Case-sensitive APFS" = not supported
  ```

  Or verify empirically, by checking whether names differing only in case collide:

  ```sh
  touch /Volumes/<name>/.ctest_a
  [ -f /Volumes/<name>/.ctest_A ] && echo OK || echo "NG (case-sensitive)"
  rm -f /Volumes/<name>/.ctest_a
  ```

- **Do not choose case-sensitive when formatting** unless there is a concrete reason.
  It is needed for things like hosting Linux source trees verbatim; media storage does not
  need it.
- **Other devices behaving normally proves nothing.** Corrupted data can propagate to other
  devices through cloud sync, yet a device that has finished ingesting it keeps running
  without symptoms. Nothing surfaces in the UI either — counting rows in the database is the
  only way to see it.
- **Do not settle for a partial fix.** Moving one library off the volume leaves every other
  Apple library on it still violating the same rule. Revisit the whole volume layout.
- **Check for case collisions before migrating.** Moving to a case-insensitive volume makes
  names that differ only in case collide, and one of them is lost. Lowercase the paths, count
  duplicates, and confirm zero before moving.
- **As a general triage rule:** when several machines share the same data and only one
  misbehaves, the cause is that machine's unique conditions, not the data. **Always include
  the filesystem format in the checklist.** Asking "does this happen in any other
  environment?" before digging into symptoms narrows the search dramatically.

## launchd periodic jobs can be delayed or dropped, and there is no way to observe it

**Symptom:** A 30-minute launchd job (launching an AI-agent batch analysis)
had one firing skipped and ran an hour late. The unprocessed backlog grew,
stretching turn time and token consumption in a positive-feedback loop. A
post-hoc AI investigation could not determine the cause.

**Why:** Since OS X 10.9, launchd timers are coalesced for power efficiency
(firings can be delayed). Additionally, while a job with the same label is
still running, the next firing is **dropped** (not queued). There is no way to
tell after the fact which happened: launchd cannot be queried for the next
fire time and keeps no record of a skip. Once the unified log retention
(days) has passed, the trail is gone entirely.

**How to apply:**
- `LegacyTimers: true` in the plist disables coalescing (precise firing at
  the cost of power efficiency).
- If a job's run time can exceed its period, do not use launchd periodic
  execution — a firing that lands mid-run is silently dropped.
- For periodic tasks where "it did not run" must be observable, use
  [task-clock](https://github.com/nlink-jp/task-clock): it evaluates cron
  itself and records every fire as scheduled-vs-actual
  (`on_time` / `queued` / `missed` with reasons), keeping launchd only for
  KeepAlive residency.
- Give resident daemons `ProcessType: Interactive` to avoid background
  throttling, and structure the timing loop as a short-interval ticker poll
  instead of one long sleep-until-next-fire timer — under App Nap /
  coalescing the worst-case delay stays bounded by the tick interval.

## Never park a launchd daemon's binary inside someone else's lifecycle

**Incident:** a daemon installed from a GUI-bundled CLI wrote
`os.Executable()` — an .app-interior path — into its plist. Merely
quitting the app took the daemon down with it, and a running task was
killed mid-flight; the work was lost.

**Why:** launchd cannot spawn a program whose path has vanished, and even a
running daemon is destroyed by whatever owns that path: the parent app
quitting, a cask upgrade replacing the .app, `brew upgrade` deleting the
Cellar, a rebuild re-signing dist/. The lifetime of the path in the plist
becomes the lifetime of the daemon.

**How to apply:**
- Have install self-copy the binary into a **daemon-owned home** (e.g.
  `data_dir/bin/<name>`) and point the plist only there. Order:
  bootout → copy → write plist → bootstrap (replacing a running binary
  gets it SIGKILLed on signature change; a failed copy must not leave a
  plist pointing at nothing).
- A durable stop is `launchctl disable` + `bootout`. A bootout without the
  disable is silently undone by RunAtLoad at the next login; and since
  launchd refuses to bootstrap a disabled service, the start path must
  `enable` first.
- To keep a daemon stop from destroying child work, record pid + process
  start time (`ps -o lstart=`) in a registry **at spawn time**, and have
  the next daemon adopt processes that are alive with a matching start
  time. Stop-time bookkeeping cannot cover a crash; command-line matching
  yields false negatives after an in-run exec and false positives after
  pid reuse — the start time survives exec and always changes with reuse.
  Reference implementation:
  [task-clock](https://github.com/nlink-jp/task-clock)'s live-run registry.
