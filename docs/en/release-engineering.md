# Release Engineering

## A fail-open step plus a verifier that only displays = a defective release with green checks

**Symptom:** The notarize script deliberately fails open (warn and exit
0 un-notarised) so contributors without credentials can still build.
The day Apple's updated developer agreement broke the profile probe,
the release went straight through and **an un-notarised zip shipped
with every check green**. The verifier *displayed* the Gatekeeper
verdict — but as `spctl … | head -2`, so only the pipeline tail's exit
status counted and a `rejected` could never stop the chain. On top of
that, the release operator compressed step output with `tail -1`,
which also hid the displayed rejection. The accident needed all three
failures stacked.

**How to apply:**
1. **Any step allowed to fail open needs a deterministic gate
   downstream.** Write a success marker (`<zip>.notarized`, touched
   only after confirming `status: Accepted`) and make the verifier
   require it. Gate on local determinism, not an online lookup (spctl's
   ticket check lags fresh submissions).
2. **`cmd | head` / `cmd | tail` return the pipe tail's exit status** —
   never put the command you are gating on inside a pipe. Separate the
   display path from the verdict path.
3. **If you compress a release step's output, grep for the success
   token.** `tail -1` relies on the last line happening to be the
   success message; on failure a hint line lands last and reads like
   success. `grep Accepted` is empty exactly when it failed.
4. **Exercise the gate in both directions**: it passes with the marker
   and fails without it. A one-direction check cannot detect a gate
   that never fires.
5. Notarising a zip does not change its bytes (bare CLI distributables
   cannot be stapled). Re-submitting the identical zip after the fact
   heals an already-published asset — the sha and the download stay
   valid. Worth remembering when deciding whether a re-upload is
   needed.


Lessons on macOS signing/notarization, release archives, and Homebrew tap
distribution. Each entry follows **symptom → why → how to apply**, extracted from
incidents on real projects. Prescriptive rules live in
[CONVENTIONS.md](https://github.com/nlink-jp/.github/blob/main/CONVENTIONS.md)
(§Release Archive Standard, §Code Signing and Notarization); this document covers
the "why" behind them and the traps that don't fit into rules.

---

## darwin binaries can be notarized even for tar.gz distributions (temp-zip pattern)

**Symptom:** Apple's notary service (`xcrun notarytool submit`) only accepts
zip / dmg / pkg containers — tar.gz cannot be submitted directly.

**Why it works:** The notary ticket is stored on Apple's servers **keyed by the
binary's CDHash** (a hash of the content). Whatever the final distribution format,
macOS looks up the ticket by CDHash at launch time.

**How to apply:** Insert "zip to a temp file → submit → discard the zip" before
packaging.

```make
dist-darwin: cross-build-darwin
	@for arch in arm64 amd64; do \
		cp dist/$(BINARY)-darwin-$$arch /tmp/$(BINARY) && \
		(cd /tmp && zip -j /tmp/$(BINARY)-notary.zip $(BINARY)) && \
		scripts/notarize-darwin.sh /tmp/$(BINARY)-notary.zip "$(NOTARY_PROFILE)"; \
		rc=$$?; \
		rm -f /tmp/$(BINARY) /tmp/$(BINARY)-notary.zip; \
		test $$rc -eq 0; \
	done
	# Packaging (tar.gz / .deb / .rpm) needs no changes — the binary is already notarized
```

- `rm -f` cleans temp files even on notarization failure; `test $$rc -eq 0`
  propagates the failure.
- The ticket is CDHash-keyed, so renaming the file or rebuilding the zip after
  submission is fine.

## notarytool 403 "required agreement missing/expired" is an agreement issue, not key expiry

**Symptom:** Previously working notarization suddenly fails across the board, and
the wrapper script reports "Keychain profile not found".

**Why:** The script pre-probed with `notarytool history` and inferred the failure
reason from the **exit code alone**. Running
`xcrun notarytool history --keychain-profile <profile>` directly revealed
`403 A required agreement is missing or has expired` — the Apple Developer Program
license agreement had been updated and the Account Holder had not yet accepted it.
Keys and profiles were perfectly valid.

**How to apply:**
- Never trust the wrapper's diagnosis; run `notarytool history` directly to see
  the real error.
- On `403 ... agreement ...`, accept the updated agreement at
  developer.apple.com/account (also check App Store Connect Agreements). It takes
  effect within minutes; no key re-registration needed.
- Fix wrappers to capture and **always display the probe's actual error**, and to
  point at agreement re-acceptance when 403/agreement is detected — a helpful
  misdiagnosis is worse than none.
- The `notarytool history` pre-probe is flaky (it can falsely fail for one arch
  right after succeeding for another). If the script path fails, bypass it with
  `notarytool submit --wait` directly.
- notarytool keychain profiles are **machine-local** (stored in the
  data-protection keychain — see the screen-lock entry below). They do not sync
  to other machines; each build machine needs its own `store-credentials`.

## notarytool credentials are unreadable while the screen is locked (they look deleted)

**Symptom:** Notarization that had just succeeded 8 times in a row on the same
machine and profile started failing uniformly with
`Error: No Keychain password item found for profile: <name>` partway through an
unattended run (2026-08, during a 10-app GUI release batch). It looks like a
deleted profile or a corrupted keychain — but the screen had simply locked.
The moment it was unlocked, everything worked again with no re-registration.

**Why:** `notarytool store-credentials` stores the credential in the
**data-protection keychain** (invisible to the legacy
`security find-generic-password` **even while unlocked** — which is what makes
"it was deleted" such an easy misdiagnosis). Data-protection keychain items
become unreadable while the console is locked, so a locked screen makes
notarytool report the item as missing. `security show-keychain-info
login.keychain-db` answering normally adds to the misdirection (login.keychain-db
is not involved).

**How to apply:**
- On this error, first check `ioreg -n Root -d1 | grep IOConsoleLocked` — if
  `Yes`, the credential is fine and **unlocking the screen is the whole fix**.
  No profile re-creation needed.
- Unattended releases (parallel agents, long batches) should either hold off
  screen lock until notarization is done, or be designed to resume the stopped
  portion later (resuming from a pushed tag with no GitHub release yet does not
  reuse a version).
- Never re-run `store-credentials` before checking the lock state — it is a
  needless API-key operation and teaches the wrong root cause.

## Binaries under cloud-synced folders can get SIGKILLed (provenance xattr)

**Symptom:** Copying a Go binary into `~/bin` under a Dropbox-synced folder makes
it die instantly with `zsh: killed` (SIGKILL). The same binary runs fine from
`dist/` (outside the synced tree).

**Why:** Dropbox's FileProvider extension attaches `com.apple.provenance` and
related xattrs the moment a file lands in its sync folder — **even for a local
`cp` on the same machine**. `com.apple.provenance` is system-protected and cannot
be removed with `xattr -d`. Gatekeeper SIGKILLs the combination of a weak
(ad-hoc / linker-signed) signature plus provenance. The behavior was tightened in
a macOS minor update, breaking a setup that used to work.

**How to apply:**
- Re-sign after placement: `codesign --force --sign - <binary>` makes it pass
  (provenance remains, but the signature is now "signed by this machine now").
  Each synced machine must re-sign — signatures don't propagate through sync.
- `cp` creates a new inode at the destination (fresh xattrs); Finder "move" on
  the same volume is a rename (inode preserved) and doesn't trigger it. `ditto`
  also works. Avoid bare `cp`.
- `spctl --add` is deprecated on current macOS ("This operation is no longer
  supported").
- **This is an environment/placement problem, not a code bug.** When a child
  process dies instantly (`read response: EOF` / `broken pipe`), first check the
  binary's exit code (137 = SIGKILL) and whether it lives under a synced folder.
- Don't add a personal-environment `install:` target (hardcoded `~/bin` cp +
  codesign) to project Makefiles. Projects end at `make build` → `dist/`;
  personal placement belongs in personal dotfiles.

## GUI .app signing is a separate pipeline from CLI

**Symptom:** CLI signing/notarization scripts assume a single Mach-O and cannot be
reused for .app bundles.

**Key points** (canonical procedure: CONVENTIONS.md §Code Signing → GUI app signing):
- **WebView-based apps (Wails / Tauri) need two entitlements at minimum**:
  `com.apple.security.cs.allow-jit` +
  `com.apple.security.cs.allow-unsigned-executable-memory`. Without them the
  Hardened Runtime kills WKWebView's JS and the frontend goes **silently blank**.
- **Native Swift / AppKit apps need no JIT entitlements** — Hardened Runtime
  alone works. HTTP to localhost is handled via `NSAppTransportSecurity` in
  Info.plist, no entitlement needed.
- **Never `cp -r`; use `ditto`**: bundle signatures live in xattrs. `cp -r` drops
  them and the app dies at launch with "Code Signature Invalid". Distribution
  zips are built with `ditto -c -k --keepParent`.
- **Signing order:** build → deep sign (`--force --deep --options runtime
  --timestamp --entitlements`) → ditto → notarize (wrap the .app in a temp zip
  for submission) → `stapler staple`. Bundle formats can be stapled, so offline
  first launch shows no dialog. Bare CLI Mach-Os cannot be stapled (online check
  applies).
- **Tauri signs inside `tauri build`** via the `APPLE_SIGNING_IDENTITY` env var.
  Tauri's built-in notarization wants its own env credentials and cannot use
  notarytool keychain profiles — leave it disabled and run xcrun notarytool
  yourself. DMG filenames use Rust triplet arch (`aarch64`), not `uname -m`
  (`arm64`). Making `package` depend on `build` re-builds and strips the staple.
- Swift Package Manager GUIs assemble the `.app` bundle structure
  (`Contents/MacOS/` etc.) manually in the Makefile, then codesign last.

## Verify release archives by downloading the published assets

**Symptom:** Build logs and local dist can look right while the published asset is
wrong. An org-wide audit (52 repos / 171 archives) caught a zip missing
README/LICENSE.

**How to apply:**
- `gh release download` the **published** assets and inspect: canonical binary
  name / `file` arch match / README.md + LICENSE bundled / darwin signature.
- **Extract .app zips with `ditto -x -k`, never `unzip`** — unzip breaks
  symlinks/xattrs and `codesign --verify --deep` then *falsely* fails with
  "a sealed resource is missing or invalid" (the original zip is fine).
- **`spctl -a -t exec` rejects bare CLI Mach-Os by design** ("the code is valid
  but does not seem to be an app") — regardless of notarization. Verify CLIs with
  `codesign --verify --strict` + `Authority=Developer ID Application` +
  identifier match. Only `.app` bundles can be assessed by `spctl`
  ("accepted, source=Notarized Developer ID").
- **Whether a bare CLI is notarized can be checked directly** with
  `codesign --test-requirement="=notarized" --verify`, which answers
  `explicit requirement satisfied`. This is stronger than falling back to the
  Authority and Identifier check above: it rejects a binary that is correctly
  signed but never notarized. Run it against the asset fetched with
  `gh release download`, not the local `dist/` copy (demonstrated on
  mcp-bridge v0.1.0, 2026-08-23).
  - **This check does fail right after notarization, though.** A bare Mach-O
    cannot be stapled, so its ticket is looked up online, and for seconds to
    minutes after `notarytool` reports `Accepted` the check still answers `code
    failed to satisfy specified code requirement(s)`. **Do not read that as a
    failed notarization** — `notarytool`'s `Accepted` is the authority, and this
    check only observes whether the ticket has been distributed yet. Running it
    against the published asset after upload builds in the wait naturally; retry
    in an until-loop if it still fails (hit on mcp-bridge v0.1.1, 2026-08-23).
- **Lightweight tags are invisible to `git describe`** (annotated-only by
  default). Derive versions with `git describe --tags` (includes lightweight).

## Executables inside release zips use the canonical name

**Symptom:** `zip -j` uses the basename of the input file as the entry name, so
zipping per-arch artifacts (`tool-darwin-arm64`) directly ships binaries with arch
suffixes, forcing users to rename on install. 32 tools needed a sweep fix.

**How to apply:** Stage-and-zip — copy the per-arch artifact to a staging dir
under its canonical name, then zip.

```make
$(eval STAGE=dist/_pkg-$(GOOS)-$(GOARCH)) \
rm -rf $(STAGE) && mkdir -p $(STAGE) ; \
cp dist/$(BINARY)-$(GOOS)-$(GOARCH)$(EXT) $(STAGE)/$(BINARY)$(EXT) ; \
zip -j $(ARCHIVE) $(STAGE)/$(BINARY)$(EXT) LICENSE README.md ; \
rm -rf $(STAGE) ; \
```

- `cp` preserves signatures for single Mach-Os (unlike .app bundles, no ditto
  needed).
- Notarization is CDHash-keyed and filename-independent, so renaming keeps it
  valid.

## CLIs must answer the `--version` flag (brew test invokes it)

**Symptom:** Tools with only a `version` subcommand fail `brew test` with
"unknown flag" because the shared formula template asserts on
`shell_output("#{bin}/<name> --version")`. `brew install` succeeds, so the
problem only surfaces after tap inclusion — four tools needed patch releases at
once.

**How to apply:**
- Wire `--version` at scaffold time (cobra: `rootCmd.Version = Version`). Keep
  the subcommand for compatibility and align both outputs.
- In regression tests, **never assign `rootCmd.Version` inside the test** — that
  grows the flag artificially and a broken binary still passes. Assert on the
  production init wiring (`rootCmd.Version != ""`).

## Run the built binary's `--version` before uploading

**Symptom:** With `VERSION ?= dev` hardcoded, `make build-all` produced release
binaries all reporting `dev`, which were uploaded as-is.

**How to apply:**
- Use `VERSION ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)`
  from the start (`?=` keeps manual override possible).
- Right after `make build-all`, run `dist/<binary>-darwin-arm64 --version` and
  confirm the expected string before uploading. Put it on the release checklist.

## Never reuse a published version number (fix-forward)

**Symptom:** Deleting and re-pushing a tag that has a GitHub Release reverts the
release to Draft and drops the "Latest" designation, requiring manual recovery.

**Why:** Once users have downloaded a binary, a different binary under the same
version number destroys the version's value as an identifier.

**How to apply:**
- Even when a fix affects published assets, don't re-release — land it in the
  next patch/minor release (fix-forward).
- Before deleting any tag, check `gh release list`. Only tags without releases
  may be re-pointed.

## Prebuilt-binary Homebrew tap practices and verification

**Symptom:** Building from source strips the Developer ID signature. The tap
installs signed+notarized release zips as-is (prebuilt).

**How to apply:**
- **Formula (Go CLI)**: `depends_on arch: :arm64` + `depends_on :macos`,
  `bin.install "<name>"`. Brew's automatic version detection doesn't work with
  version-in-the-middle asset names, so hardcode `url` and declare `version`.
  `brew style`'s ComponentsOrder requires `url` before `version`, so `#{version}`
  interpolation in `url` is not possible.
- **Cask (GUI .app)**: `depends_on macos: :big_sur` (string forms get corrected
  to symbols by style), `zap trash: [...]`.
- **The cask is the true notarization test**: brew strips quarantine xattrs for
  formulae but **keeps them for casks** → `spctl -a -t exec` performs the full
  Gatekeeper assessment. "accepted, source=Notarized Developer ID" on a clean
  machine is decisive proof the signature survived.
- **Without a clean VM**: download the published zip, attach
  `com.apple.quarantine` yourself, then `codesign --verify --strict` and execute
  the quarantined binary. A quarantined notarized binary that runs has passed
  Gatekeeper's online check.
- **Pre-publication testing needs no GitHub**: clone the tap working copy into
  `$(brew --repository)/Library/Taps/<org>/homebrew-tap` and
  `brew install <org>/tap/<name>` works (urls point at public release assets).
  `brew audit --tap` skips casks unless a Casks/ directory exists.
- brew over ssh is not on PATH (`/opt/homebrew/bin` comes via `.zprofile`) —
  prefix commands with `eval "$(/opt/homebrew/bin/brew shellenv)"`. Cask installs
  can use `--appdir="$HOME/Applications"` to avoid the sudo prompt (which hangs
  non-interactive ssh), but **`--appdir` is install-only — uninstall rejects it**
  (it resolves from the receipt).
- Cleanup order: uninstall → untap (untap refuses while artifacts remain
  installed).

## Re-signing a dist/ binary kills daemons running from it

**Symptom:** `make build` overwrote and re-signed `dist/<binary>`, and the
resident daemon launched from that binary was killed by macOS — data recording
silently stopped for over 30 minutes.

**Why:** macOS kills a running process when its binary's code signature changes.
Without a LaunchAgent registration, `KeepAlive` cannot restart it either.

**How to apply:**
- Launch resident daemons from the **.app-bundled path**
  (`Contents/Resources/<binary>`), not from `dist/`. It is unaffected by
  `make build` and recovers via the LaunchAgent's `KeepAlive`.
- Around any build work, check the daemon's status (last-sample time). A long gap
  may be your own build's doing.

## For a package with a bundled payload, size gives the lie away first

**Symptom:** a release that bundles a CLI binary inside a GUI app was built with
`make package` straight after `make clean`. The bundling step was **silently
skipped and the result still passed notarization**. The zip was 1.2 MB where a
correct one is about 10 MB, which is how pre-upload verification caught it.

**Why:** the bundling step guarded on `[ -x "$CLI_BIN" ]` and merely warned when
the file was absent. `make clean` had removed the plainly-named binary, and the
`build-all` target that `package` depends on only produces
platform-suffixed names. Neither signing nor notarization detects missing
*contents*.

**How to apply:**
- **Always extract the archive and inspect it before uploading**: the payload is
  present, the executable answers `--version`, and the signature checks out
  (`spctl --assess`) — on the real artifact, not the build tree.
- Payload size is the cheapest sanity signal there is. An order-of-magnitude
  change deserves suspicion.
- If a dependency step "warns and continues" when something is missing, make it
  **fail on the release path**. Convenience during development becomes a
  shipping accident unchanged.

## Pin the version string explicitly when building something to bundle

**Symptom:** a single Makefile line changed after tagging, and `git describe`
started returning `v0.1.0-1-g<sha>`, so the **bundled binary reported a
non-release version**.

**Why:** with a version derived from `git describe --tags`, any commit after the
tag changes the string — even one that touches nothing compiled.

**How to apply:** build bundled payloads with the version stated outright
(`make build VERSION=vX.Y.Z`), after confirming with
`git diff --stat <tag>..HEAD` that nothing compiled actually changed.

## A status written at scaffold time is not updated by releasing

**What happened:** three tools were found whose READMEs said "pre-release" or
"not yet released". All three had shipped. One had **shipped four times, had a
Homebrew formula, and printed that very `brew install` command two lines below
the banner**. Another said "not yet released — once published:" and then gave a
brew command that already worked.

**Why:** the release checklist looks at the changelog, the version, the
signature, and the archive. **Nobody looks at prose written at scaffold time.**
And "not released yet" is *true* when written — it only becomes false at the
first release, which is exactly the moment its author is thinking about something
else. A related case: a catalog entry kept advertising a behaviour that a later
measurement had disproved.

**How to apply:**

- **Do not state status in prose.** Whether something has shipped is already
  conveyed by whether there are install instructions. Do not write a version
  number either — `git tag` is the single source.
- If you must state it, put it only in a file the **release procedure itself
  touches** (the changelog). A banner at the top of a README is in nobody's
  checklist.
- **This is mechanically detectable**: "has at least one release AND the README
  matches `not yet released|pre-release`" is a grep plus a `gh release list`, and
  belongs in whatever org-wide check you already run.
