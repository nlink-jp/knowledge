# Changelog

## 2026-08-03

- 1 entry from a launch crash that reached two shipped macOS apps:
  - **macos-gui** (1): never use SwiftPM's `Bundle.module` inside an `.app` —
    it looks beside the bundle root, not in `Contents/Resources`, and falls
    back to a compile-time absolute `.build` path, so it resolves only on the
    machine that built it. Includes the release-verification step that
    reproduces a foreign machine.

- 15 entries from building a menu-bar sensor-monitoring app over a metered
  third-party API:
  - **macos-gui** (7): MenuBarExtra pushing a height onto its content; ForEach id
    uniqueness across a whole List; timers needing the `.common` run-loop modes;
    elapsed-time labels needing a TimelineView; not disabling a control on an
    ambiguous status; asking for OS permission at the moment of intent; changes
    in a second ObservableObject not reaching views that do not observe it.
  - **development-process** (2): building the budget into the design when the
    API is metered and exhaustion looks like an auth failure; preventing
    duplicate workers with a conditional no-op rather than a protocol.
  - **testing** (2): green tests proving nothing when the expectation itself is
    wrong; verifying a GUI on the assumption that what you can see and what runs
    are independent.
  - **release-engineering** (2): payload size giving away a silently skipped
    bundling step; pinning the version string when building something to bundle.
  - **config-and-io** (2): searching both config conventions on macOS;
    long-format storage making re-import idempotent.
  - **security** (1): putting the restraint in the client when the credential is
    more powerful than the use.

## 2026-08-02

- Initial compilation (ADR-015): 13 themed documents in Japanese and English,
  compiled from ~100 engineering-knowledge memories accumulated across
  nlink-jp projects.
