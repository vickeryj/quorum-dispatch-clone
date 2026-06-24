# ADR 0018: Packaging & install for the Rust sb stack (sb + sbx + plugins)

**Status:** Proposed — **gated** by stress-test findings ([tbd/0002](../tbd/0002-packaging-stress-test-findings.md)); blocked on one design decision (the canonical/merged shape of `plugins/core`) before A/B execute.
**Date:** 2026-06-16
**Relates-to:** the retired TS switchboard `sb bootstrap`/`build-dist` design (ADR-0002 of that repo, now archived); sb-rust's existing `bootstrap` + `update` verbs; the existing Stage-1 Homebrew work (`packaging/homebrew/sb.rb`, "A7"); stress-test gate ([tbd/0002](../tbd/0002-packaging-stress-test-findings.md)).

## Context

The Rust stack is three independently-versioned artifacts in three repos:

- **`sb`** — the engine binary (`private-org/sb-rust`).
- **`sbx`** — the extension binary: obligation/continuity verbs (`private-org/sbx`).
- **plugins** — the work-model content: the `core` plugin pack + substrate (`private-org/plugins`).

There is currently **no install story**. The whole `bootstrap`/`update`/marketplace machinery lived in
the TypeScript switchboard, which has been archived. On the Rust side, `sb` is installed by hand
(`cargo build --release` + `cp`), `sbx` likewise, and the content is built ad hoc. There is no way for
someone to "get a known-good, version-matched set of all three."

Two facts shape the decision:

1. **`sb` already has the bones.** It carries a `bootstrap` verb (sets up `~/.sb` + native relay
   registration + shell init) and an `update` verb that already detects its install channel
   (*Homebrew or cargo*). What `bootstrap` does **not** yet do is pull/build `sbx` or the plugins.
2. **The content has two homes today.** `sbx` still carries `core/`, `code/`, `lab/` plus a
   `build-dist.sh` and build tooling — *and* `private-org/plugins` already exists, is active, and has
   its own `core/`. This duplication has to be resolved before "pin the plugins" means anything.

## Decision

**`sb` is the single entry point and the carrier of the pin. You choose one version (the `sb` tag);
that determines the other two.**

1. **One entry point, no fourth package.** Distribution is `cargo install --git <sb-rust> --tag vX`.
   No separate meta-installer — `sb` already *is* the carrier (`bootstrap`/`update`). rustup is
   irrelevant (it manages toolchains, not apps).
2. **Three pins, expressed as one.** The user pins only `sb` (the cargo `--tag`/`--rev`). `sb` carries
   a baked-in data manifest (`extensions.toml`, compiled via `include_str!`) naming the exact `sbx`
   ref and plugins ref blessed for that `sb` version. That manifest is the lockfile and the single
   source of truth for "what versions go together."
3. **`sbx` becomes engine-only; plugins is core's sole home.** Strip `core/`/`code/`/`lab/` + the
   build tooling out of `sbx` ("kill it with fire"). The work-model content and its build tooling live
   only in `private-org/plugins`.
4. **`sb bootstrap` grows a consent-gated cascade.** After its existing steps it offers, in order:
   "install `sbx`?" → install the pinned `sbx`; then "install the core plugin pack?" → clone
   plugins@pinned-ref, build it, wire it into `~/.sb`. Layered: engine-only, +sbx, +plugins are all
   valid resting states. The versions come from the manifest, so the user never types a version for
   `sbx` or plugins.
5. **Stay private; cargo over SSH.** `cargo install --git` and bootstrap's `git clone` use the
   machine's SSH/git auth, so all three repos stay **private**. No public flip required for the
   immediate term.

### Why this, and not the alternatives

- **Why one pin in `sb`, not three the user juggles:** the failure we are designing against is an
  *incompatible combo* — `sb` vX against an `sbx`/plugins it was never tested with. Collapsing the
  choice to a single version makes "matched set" the default and "mismatch" something you have to go
  out of your way to do. It mirrors the archived TS ruling "the binary carries the plugins."
- **Why no fourth package:** a meta-installer would re-implement `sb`'s carrier role with a new moving
  part and its own version. `sb` is already the thing you install first and the thing that knows the
  rest.
- **Why cargo/private now, Homebrew/public later:** cargo-over-SSH needs no release infra and keeps
  the repos private — it works today on Pete's machines and the agent fleet. Homebrew earns its keep
  only when shipping *prebuilt* bottles to toolchain-less strangers, which drags in CI, macOS
  signing/notarization (the `Killed: 9` we hit installing a copied binary is a small preview), and a
  public flip. Sequencing loses nothing: `sb update` already detects Homebrew-or-cargo, so v2 slots in
  without rework.
- **Why fix the two-homes problem first:** until `sbx` stops carrying `core/`/`code/`/`lab/`, "pin the
  plugins repo" is ambiguous and bootstrap could deploy stale content. It is a hard prerequisite, not
  a cleanup nicety.

## Consequences

- A clean-machine flow becomes: `cargo install --git …sb-rust --tag vX` (binary lands in
  `~/.cargo/bin`, on PATH) → `sb bootstrap` → answer two prompts → matched `sbx` + plugins in `~/.sb`.
- Cutting content out of `sbx` is a breaking repo change; anything that referenced `sbx`'s
  `build-dist.sh` or `core/code/lab` must be repointed at the plugins repo first.
- `extensions.toml` must be validated at `sb` build time (a tag it names that doesn't exist or
  doesn't build is a release-blocking error, never a runtime surprise for the user).
- Bootstrap must stay idempotent, partial-safe, and non-interactive-safe (the existing steps already
  are); the new steps must hold the same bar and fail *loudly* when SSH/toolchain is absent.
- **Out of scope (v2):** Homebrew tap, prebuilt GitHub Releases binaries (macOS arm64, Linux arm64,
  Linux x86_64), CI/CD, and making the repos public. The rollout decomposition lives in
  `doc/tbd/0001-packaging-rollout-plan.md`.
