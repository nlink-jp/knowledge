# Build & Packaging

Lessons on Makefiles, cross-compilation, .gitignore, and CI policy. Each entry
follows **symptom → why → how to apply**. Baseline rules (`make build` → `dist/`,
versions from `git describe`) live in CONVENTIONS.md §Build Conventions.

---

## Windows cross-compiles of CGO + upstream prebuilt static libs need UCRT + ranlib

**Symptom:** Cross-compiling go-duckdb (bundled prebuilt `libduckdb.a`) to
Windows from a Debian-based container with the default `gcc-mingw-w64-x86-64`
fails through a chain of linker errors: `cannot find -lstdc++` →
`undefined reference to pthread_*` → C++ ABI mismatch (`_Mbstatet` name
mangling) → `archive has no index`.

**Why:**
- **The root cause is the prebuilt library being compiled against the UCRT
  (Universal C Runtime) ABI.** Windows 10+ standardizes on UCRT; the default
  mingw (MSVCRT-based) mangles C++ names differently.
- Additionally, Debian's `gcc-mingw-w64-ucrt64` package ships `.a` files lacking
  archive indexes (a packaging issue), so `ranlib` must rebuild indexes across
  the toolchain.

**How to apply:** Skip the symptom-by-symptom chase; start with the UCRT
toolchain + ranlib-all:

```makefile
build-windows:
	$(CONTAINER) run --rm -v "$(CURDIR):/workspace:z" -w /workspace $(GO_IMAGE) \
		bash -c 'apt-get update -qq && apt-get install -y -q gcc-mingw-w64-ucrt64 g++-mingw-w64-ucrt64 \
			&& find /usr/lib/gcc/x86_64-w64-mingw32ucrt /usr/x86_64-w64-mingw32ucrt -name "*.a" -exec x86_64-w64-mingw32ucrt-ranlib {} + \
			&& GOOS=windows GOARCH=amd64 CGO_ENABLED=1 \
			CC=x86_64-w64-mingw32ucrt-gcc CXX=x86_64-w64-mingw32ucrt-g++ \
			go build -ldflags "-X main.version=$(VERSION)" -o $(DIST_DIR)/$(BINARY)-windows-amd64.exe .'
```

General principle: **when using a prebuilt C++ static library for Windows, first
determine which C runtime (UCRT / MSVCRT) it was built against** (usually visible
in upstream build instructions or CI workflows).

## Never put binary names in .gitignore — it swallows cmd/<name>/

**Symptom:** Writing the binary name directly into `.gitignore` also excluded the
`cmd/<name>/` directory, leaving main.go untracked — code was lost on multiple
projects.

**How to apply:**
- Unify build output under `dist/` and list only `dist/` in `.gitignore`. No
  binary-name patterns at all.
- Do keep standard junk exclusions: macOS junk (`.DS_Store` `._*` …), editor
  leftovers (`*.swp` `*~`), and `.claude/`. "No binary names" and "standard junk
  yes" are compatible.

## The decision to release from local builds (no GitHub Actions)

**Symptom:** CI/CD-ifying releases (GitHub Actions) keeps getting considered or
proposed; local machine builds are maintained deliberately.

**Why (recorded as a decision rationale):**
1. **macOS runner cost**: Actions bills macOS at 10× Linux. Even a personal-OSS
   release cadence easily exceeds the free tier.
2. **Where signing keys live**: the Developer ID certificate and notary API key
   should exist only in the local Keychain. GitHub Secrets widens the blast
   radius on leak, the key-to-runner path is itself attack surface, and Apple's
   terms envision sole keyholder control of the signing identity.

**How to apply:**
- Reduce human error by making `make package` a one-shot
  cross-compile + codesign + notarize + zip (scripting replaces CI).
- For well-meaning CI PRs (adding release workflows), acknowledge the motivation,
  explain this policy, and close politely. Even "lint/test-only CI" is declined —
  exceptions blur the policy; local `make check` covers it.

---

## Pin vendored dependencies to immutable version tags, never rolling ones

**What happened:** Vendoring sherpa-onnx as a submodule for a static build failed at
the point where cmake fetches the ONNX Runtime archive: the downloaded zip was intact
(`unzip -t` passed) but its SHA256 did not match the hash upstream had pinned. The
submodule HEAD tracked master, near a tag called `xcframework`.

**Why:** `xcframework` is a **rolling tag** upstream re-uploads assets to (the GitHub
release page shows assets with mixed dates), and the commit was mid-flight — "set
onnxruntime version to 1.27.1". The problem was **depending on a mutable reference**,
not an upstream defect. The ordinary version tag `v1.13.4` matched its hash exactly.

**How to apply:** Pin submodules to **`vX.Y.Z`-style immutable tags**. Rolling tags
(`latest`, `nightly`, feature-named tags like `xcframework`) and arbitrary master
commits are not reproducible, because a third party can change what they point at.

When upstream fetches a prebuilt archive during configure, it also helps to **extract
the URL and hash from upstream's own definition, fetch it yourself, and verify the
hash before putting it in place** — which sidesteps a cmake downloader that fails on
some toolchains:

```makefile
# Avoid parentheses inside $(shell ...): a regex containing one breaks make's parser
ORT_URL  = $(shell grep -m1 'set.onnxruntime_URL' $(ORT_CMAKE) | cut -d'"' -f2)
ORT_HASH = $(shell grep -m1 'set.onnxruntime_HASH' $(ORT_CMAKE) | cut -d'"' -f2 | cut -d= -f2)
$(ORT_ZIP):
	@curl -fsSL --retry 3 -o $@ "$(ORT_URL)"
	@echo "$(ORT_HASH)  $@" | shasum -a 256 -c - || (rm -f $@; exit 1)
```

---

## "Pin the version" is sometimes not a tamper defence — inventory the exposure first

**What happened:** Work began from the position "supply-chain attacks are
worrying, so pin the submodule to a release tag". Taking an inventory showed
**the place being pinned was not the problem, and the hole was somewhere else.**

**Why:**
- **A git submodule is already fixed by commit SHA, and a SHA is
  content-addressed** — stronger against tampering than a tag name, which can be
  moved. Moving to a release tag mainly buys **auditability** (matching against
  advisories), not tamper resistance.
- Worse, moving "back" to the release tag would have **dropped known security
  fixes**: the unreleased commits past the last tag included a heap
  out-of-bounds read and a stack-buffer-overflow, both on paths the tool feeds
  untrusted input to.
- The actual hole was that **models downloaded at runtime were verified by size
  only**. Size is trivially preserved by anyone able to substitute the file, and
  those files are then handed to a C++ parser.

**How to apply:** Do not start from a policy. **List everything fetched and write
down when it is fetched and what verifies it.**

| Fetched | When | Verified by |
|---|---|---|
| submodule | build | git SHA (content-addressed) |
| prebuilt archive | build | SHA256 |
| **models / data** | **runtime** | **← the one usually missing** |

Then:
- Separate the goals: **immutability comes from the SHA, a release tag is for
  auditability.** If a fix is unreleased, staying on the commit that has it is
  the safer choice.
- **Pin and verify a hash** for anything downloaded. Most sources publish one
  (Hugging Face exposes `lfs.sha256` through its API).
- **Close the "file already present, skip the download" fast path.** A size-only
  check there is the most dangerous kind: it reads as verification.
- Verify **before the rename**, so a failing file never exists at the real path.
- Test the case that matters: **same length, different bytes.**

**Side benefit:** collecting the hashes revealed that two catalog entries shipped
**byte-identical files** — a third-party mirror redistributing the authors' build
under a different version number — and that it was the *default*. **Take defaults
from the upstream authors.** Hash collection doubles as an inventory.

## A module cache inside the repository gets caught by recursive rewrites — find it with `go mod verify`, rebuild with `go clean -modcache`

**Symptom:** a docs-only change failed `make test` with `no required module provides package github.com/spf13/cobra`,
though `go.mod` required it and the cache held it (2026-09-22, scat).

**Cause:** the Makefile pointed `GOMODCACHE` at `.cache/go-mod` inside the repository (a sandbox workaround). A
past accident that ran a recursive formatter over the workspace had rewritten the dependency sources in it.
`go mod verify` named them: `dir has been modified` (`cobra`, `x/sys`). `.cache/` is ignored by git, so no diff
showed it.

**How to apply:**
- For a module-resolution failure that makes no sense, run `GOMODCACHE=<that path> go mod verify` first.
- Rebuild with `GOMODCACHE=<that path> go clean -modcache`. The cache is created read-only, so `rm -rf` (even
  inside `make clean`) stops half-way with `Directory not empty`.
- If the cache lives in the repository, make sure formatters and bulk replacements cannot reach it.


### Keep macOS file metadata out of Linux release tarballs — `COPYFILE_DISABLE` is not enough

**Symptom:** A four-platform Go CLI release (2026-09-22) built with macOS
`tar -czf` contained `._LICENSE`, `._README.md` and `._<binary>` alongside the
intended license, README and binary. Build, signing and notarization had passed;
only inspection of archive entry names exposed the extra files.

**Why:** macOS tar carries extended attributes into the archive in two
different shapes, and one setting stops only the first. `COPYFILE_DISABLE=1`
suppresses the AppleDouble entries (`._name`), but bsdtar still writes each
attribute as a pax header — `LIBARCHIVE.xattr.*` and `SCHILY.xattr.*`, which
`tar -tzf` does not list and GNU tar reports as unknown keywords on extraction.
Measured the next day on the same project (2026-09-23): after the
`COPYFILE_DISABLE=1` fix shipped, the archives still carried
`com.apple.provenance` and, because the tree sits in a synced folder,
`com.dropbox.attrs`. A staging directory of ordinary files does not prove the
archive holds the same set, and neither does the flag that was supposed to fix it.

**How to apply:** Pass `--no-xattrs` to `tar` (both bsdtar and GNU tar accept
it); keep `COPYFILE_DISABLE=1` for the AppleDouble path. Then **check the
archive, not the flag**: make the release gate list the entries and require
exact equality with the distribution contract (binary, README, license), refuse
any `._*`, `PaxHeader` or `__MACOSX` entry, and grep the decompressed stream for
`LIBARCHIVE.xattr` / `SCHILY.xattr`. A packaging claim that nobody's gate can
fail is how the first fix shipped believed. Signature/notarization checks and
archive-content checks are independent; neither substitutes for the other.
