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


### Keep macOS file metadata out of Linux release tarballs — it takes both `COPYFILE_DISABLE=1` and `--no-xattrs`

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
The converse holds too: `--no-xattrs` without `COPYFILE_DISABLE=1` drops the pax
headers but still writes the `._` members. Extracted with GNU tar 1.35 (2026-09-23):
neither setting — warnings and stray `._` files; `COPYFILE_DISABLE=1` alone —
warnings; `--no-xattrs` alone — silent, with stray `._` files; both — clean.

**How to apply:** Archive with `COPYFILE_DISABLE=1 tar --no-xattrs` — each
setting stops one of the two shapes (both bsdtar and GNU tar accept
`--no-xattrs`). Then **check the archive, not the flag**: make the release gate
list the entries and require exact equality with the distribution contract
(binary, README, license), refuse any `._*`, `PaxHeader` or `__MACOSX` entry,
and check that no entry's **pax headers** hold a key starting with
`LIBARCHIVE.xattr.` / `SCHILY.xattr.` (`pax_headers` in Python's `tarfile`).
Do not grep the decompressed stream: it matches file text too. A bundled
CHANGELOG that named the keywords got a clean archive refused at the release
gate (2026-09-23). List
with `tar --options 'tar:!mac-ext' -tzf`: macOS bsdtar folds `._` members into
the entry they describe when it lists, so a plain `tar -tzf` shows a clean list
and a `._` check over it can never fire. Compare the sorted list in the C locale
(`LC_ALL=C sort`); under a UTF-8 locale `abuse-lookup` sorts before `LICENSE`,
and a literal expected string refuses a correct archive. Prove the gate with a
fixture per shape, the `--no-xattrs`-only one included, read back first with a
reader that does not fold (Python's `tarfile`). A packaging claim that nobody's
gate can fail is how the first fix shipped believed. Signature/notarization checks and
archive-content checks are independent; neither substitutes for the other.

## C safety layers do not fail on findings by default — positive controls read the build rules that actually run

**Symptom:** timeout's five layers (warnings, static analysis, `-fbounds-safety`, ASan/UBSan, input stress) measured
with Apple clang 21:
- `clang --analyze -Werror` exits 0 with findings; only `-Xanalyzer -analyzer-werror` fails.
- UBSan reports and carries on, exit 0; it needs `-fno-sanitize-recover=all`.
- Apple's SDK treats `_FORTIFY_SOURCE` as at most 2 (3 has no effect; read in `secure/_common.h`).
- Apple clang has no libFuzzer (`-fsanitize=fuzzer`).
- The layers catch different things: a `memcpy` past an array passed warnings and static analysis and only ASan
  caught it; index 3 into a 3-element array was trapped by `-fbounds-safety` and read silently by a normal build.
- `-fbounds-safety` accepts `main` taking `char *__null_terminated *__counted_by(argc) argv`; the single call to
  `execvp` needs `__unsafe_forge_null_terminated`. Passing a local array to a `const char *` parameter, or
  `SIG_IGN` through a function-pointer parameter, is rejected (assigning `sa_handler` directly and a counted
  `write` are accepted).
- A positive control that checks only the flag variables passes when a rule loses the flag. Reading the rules with
  `$(MAKE) -n` also reads a different Makefile unless given `-f $(firstword $(MAKEFILE_LIST))`.

**How to apply:** give each layer its failing flags explicitly; at the start of `make test`, prove each layer fails,
with its diagnostic text, on a file with a planted bug; check the flags are in the rules through `make -n` of the
Makefile that is running.

## A cgo binary's minimum macOS is set by `MACOSX_DEPLOYMENT_TARGET` — CGO_CFLAGS/CGO_LDFLAGS alone leave Go's runtime at the build machine's OS

**Symptom:** To give an MCP server that calls Core Text through cgo (report-studio-mcp) a
minimum OS of 26.0, `-mmacosx-version-min=26.0` was passed in `CGO_CFLAGS` and `CGO_LDFLAGS` on
a macOS 27.2 build machine. The linker warned of objects "built for newer macOS": Go's runtime
cgo objects were still built for the build machine's OS (measured, 2026-11-03).

**Why:** The flags reach only your own packages' C compile and link. When the Go toolchain builds
the runtime's cgo part it reads the `MACOSX_DEPLOYMENT_TARGET` environment variable (inferred
from the observed behaviour; the toolchain source was not read).

**How to apply:**
- Put `export MACOSX_DEPLOYMENT_TARGET := <version>` in the Makefile and rebuild everything. In
  the case above the warnings went to 0.
- Check `minos` with `vtool -show-build <binary>`, and look at it on every release.
- The variable takes effect only when the external linker runs (some package imports "C"). The
  same build without cgo reported `minos 13.0` (measured).
- The Homebrew formula template declares no minimum macOS, so on an older OS the tool installs
  and then fails to start. State the minimum OS in the README.

## A local patch to a vendored dependency needs applying, rebuilding and verifying checked separately — in a nested submodule none of the defaults is enough

**Symptom:** An image-generation CLI (image-forge) patched one function in a submodule of the C++
library it vendors. The patch was kept as a file, applied before the build, and checked for
presence. An independent review found three gaps in that arrangement (each measured in a throwaway
repository and with `make -n`, 2026-11-13).

1. **The library was rebuilt "when older than the patch file".** Moving the submodule pointer does
   not change the patch file's time. The library is not rebuilt, and the old one can be linked as
   it is.
2. **The check looked only at the patched directory, against the index.** A hand edit in the outer
   library's sources, a local commit in the inner checkout and an untracked file all passed.
3. **`ignore = dirty` in `.gitmodules` also removed `-dirty` from the version.** The setting is
   needed, because the patch makes the working copy differ from the pinned commit at all times.
   With it, `git describe --dirty` returns a clean version however the vendored tree is edited.

**Why:** All three come from building the arrangement around "apply the patch" alone. Make's file
times know nothing of git pointers. `git diff` compares the working tree with the index by default.
`ignore = dirty` quiets status output, and does not let you choose what goes quiet.

**How to apply:**
- Run the vendored library's build every time and let its own build system (cmake, say) decide
  what is stale. An up-to-date tree costs a few seconds. Apply the patches just before it, every
  time (a no-op when they are applied).
- Verify the vendored checkout and every submodule inside it. For each: it sits on the commit its
  parent records (the line prefix of `git submodule status --recursive`), it has no untracked
  file, and its difference from the pinned commit (`git diff HEAD --numstat`) equals what its
  patches change. A directory with no patch must show no difference.
- Once `ignore = dirty` is in use, treat the version string as no longer reflecting the vendored
  tree, and put the check above on every path that produces a binary.
- Write the order for moving the pointer (remove the patches, move it, build) at the top of the
  verification script. `git submodule update` with a patch applied is either refused or carries
  the edit across unexamined.
- If the check compares only names and line counts, say so. The document that said "a hand edit
  cannot be linked into a binary" claimed more than the check implemented.

## Working around a dependency's defect through its settings leaves what the settings do not reach — build an input that meets exactly one condition and measure

**Symptom:** After updating the same library, the GPU backend aborted on models holding one weight
format (one-byte FP8). The library has a setting that converts tensors matching a name rule to
another type as they are read, and that was used to convert the affected files. Every catalog model
worked. The independent pre-release review read in the library's source that some tensors are
exempt from any rule. No real file of that kind was at hand. A copy of an existing checkpoint with
a single tensor rewritten in that format aborted on the new version and rendered on the released
one (measured). That settled it as a regression; the settings workaround was dropped and the
function at the cause was patched.

Before that, the reason for the abort itself was invisible. The library discards every log line,
its backend's errors included, unless a receiver is registered. Only with one registered did the
name of the missing kernel appear (measured).

**Why:** A setting reaches only as far as the dependency arranged. That extent is in the
dependency's source, and when your own test subjects (the catalog models) happen to sit inside it,
every check passes.

**How to apply:**
- When avoiding a dependency's defect through a setting, read in its source the conditions under
  which the setting is **not applied**: exemption lists, branches on type, alternative paths.
- If nothing at hand meets such a condition, make it. A copy of an existing input with one element
  rewritten can often be produced by rewriting the header and appending data. Run it through both
  the new version and the released one; if only one fails, it is a regression.
- When something "just crashes with no reason", first check whether the dependency is discarding
  its logs. A library that needs a callback registered says nothing without one.
- Once the setting is known not to reach, switch to fixing the place of the cause (a patch, an
  upstream report). Enumerating the unreached cases on your side and refusing them means carrying
  a copy of the dependency's exemption list.
