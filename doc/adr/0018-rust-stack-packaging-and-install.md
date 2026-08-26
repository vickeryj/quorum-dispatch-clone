# ADR 0018: Packaging & install for the Rust qd stack (qd + qb + plugins)

**Status:** Proposed — **gated** by stress-test findings ([tbd/0002](../tbd/0002-packaging-stress-test-findings.md)); blocked on one design decision (the canonical/merged shape of `plugins/core`) before A/B execute.
**Date:** 2026-06-16
**Relates-to:** the retired TS switchboard `qd bootstrap`/`build-dist` design (ADR-0002 of that repo, now archived); qd-rust's existing `bootstrap` + `update` verbs; the existing Stage-1 Homebrew work (`packaging/homebrew/quorum-dispatch.rb`, "A7"); stress-test gate ([tbd/0002](../tbd/0002-packaging-stress-test-findings.md)).

## Context

The Rust stack is three independently-versioned artifacts in three repos:

- **`qd`** — the engine binary (`vickeryj/quorum-dispatch-clone`).
- **`qb`** — the extension binary: obligation/continuity verbs (`vickeryj/qb`).
- **plugins** — the work-model content: the `core` plugin pack + substrate (`vickeryj/plugins`).

There is currently **no install story**. The whole `bootstrap`/`update`/marketplace machinery lived in
the TypeScript switchboard, which has been archived. On the Rust side, `qd` is installed by hand
(`cargo build --release` + `cp`), `qb` likewise, and the content is built ad hoc. There is no way for
someone to "get a known-good, version-matched set of all three."

Two facts shape the decision:

1. **`qd` already has the bones.** It carries a `bootstrap` verb (sets up `~/.quorum/dispatch` + native relay
   registration + shell init) and an `update` verb that already detects its install channel
   (*Homebrew or cargo*). What `bootstrap` does **not** yet do is pull/build `qb` or the plugins.
2. **The content has two homes today.** `qb` still carries `core/`, `code/`, `lab/` plus a
   `build-dist.sh` and build tooling — *and* `vickeryj/plugins` already exists, is active, and has
   its own `core/`. This duplication has to be resolved before "pin the plugins" means anything.

## Decision

**`qd` is the single entry point and the carrier of the pin. You choose one version (the `qd` tag);
that determines the other two.**

1. **One entry point, no fourth package.** Distribution is `cargo install --git <qd-rust> --tag vX`.
   No separate meta-installer — `qd` already *is* the carrier (`bootstrap`/`update`). rustup is
   irrelevant (it manages toolchains, not apps).
2. **Three pins, expressed as one.** The user pins only `qd` (the cargo `--tag`/`--rev`). `qd` carries
   a baked-in data manifest (`extensions.toml`, compiled via `include_str!`) naming the exact `qb`
   ref and plugins ref blessed for that `qd` version. That manifest is the lockfile and the single
   source of truth for "what versions go together."
3. **`qb` becomes engine-only; plugins is core's sole home.** Strip `core/`/`code/`/`lab/` + the
   build tooling out of `qb` ("kill it with fire"). The work-model content and its build tooling live
   only in `vickeryj/plugins`.
4. **`qd bootstrap` grows a consent-gated cascade.** After its existing steps it offers, in order:
   "install `qb`?" → install the pinned `qb`; then "install the core plugin pack?" → clone
   plugins@pinned-ref, build it, wire it into `~/.quorum/dispatch`. Layered: engine-only, +qb, +plugins are all
   valid resting states. The versions come from the manifest, so the user never types a version for
   `qb` or plugins.
5. **Stay private; cargo over SSH.** `cargo install --git` and bootstrap's `git clone` use the
   machine's SSH/git auth, so all three repos stay **private**. No public flip required for the
   immediate term.

### Why this, and not the alternatives

- **Why one pin in `qd`, not three the user juggles:** the failure we are designing against is an
  *incompatible combo* — `qd` vX against an `qb`/plugins it was never tested with. Collapsing the
  choice to a single version makes "matched set" the default and "mismatch" something you have to go
  out of your way to do. It mirrors the archived TS ruling "the binary carries the plugins."
- **Why no fourth package:** a meta-installer would re-implement `qd`'s carrier role with a new moving
  part and its own version. `qd` is already the thing you install first and the thing that knows the
  rest.
- **Why cargo/private now, Homebrew/public later:** cargo-over-SSH needs no release infra and keeps
  the repos private — it works today on Pete's machines and the agent fleet. Homebrew earns its keep
  only when shipping *prebuilt* bottles to toolchain-less strangers, which drags in CI, macOS
  signing/notarization (the `Killed: 9` we hit installing a copied binary is a small preview), and a
  public flip. Sequencing loses nothing: `qd update` already detects Homebrew-or-cargo, so v2 slots in
  without rework.
- **Why fix the two-homes problem first:** until `qb` stops carrying `core/`/`code/`/`lab/`, "pin the
  plugins repo" is ambiguous and bootstrap could deploy stale content. It is a hard prerequisite, not
  a cleanup nicety.

## Consequences

- A clean-machine flow becomes: `cargo install --git …qd-rust --tag vX` (binary lands in
  `~/.cargo/bin`, on PATH) → `qd bootstrap` → answer two prompts → matched `qb` + plugins in `~/.quorum/dispatch`.
- Cutting content out of `qb` is a breaking repo change; anything that referenced `qb`'s
  `build-dist.sh` or `core/code/lab` must be repointed at the plugins repo first.
- `extensions.toml` must be validated at `qd` build time (a tag it names that doesn't exist or
  doesn't build is a release-blocking error, never a runtime surprise for the user).
- Bootstrap must stay idempotent, partial-safe, and non-interactive-safe (the existing steps already
  are); the new steps must hold the same bar and fail *loudly* when SSH/toolchain is absent.
- **Out of scope (v2):** Homebrew tap, prebuilt GitHub Releases binaries (macOS arm64, Linux arm64,
  Linux x86_64), CI/CD, and making the repos public. The rollout decomposition lives in
  `doc/tbd/0001-packaging-rollout-plan.md`.
