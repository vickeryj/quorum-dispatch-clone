# Installing the qd stack (qd + qb + plugins)

One version of `qd` determines the matched `qb` + work-model plugin. You pin `qd`; `qd` carries the
rest in [`../extensions.toml`](../extensions.toml).

## Prerequisites
- A Rust toolchain (`cargo`).
- SSH access to the private `vickeryj` GitHub repos (cargo pulls over SSH).
- The `claude` CLI on `PATH` (for the work-model plugin install).

## 1. Install the engine (pinned)
`qd-rust` is a **virtual workspace**, so the install command needs a package selector — and it
needs **two**, because `qd` no longer works alone:

```bash
CARGO_NET_GIT_FETCH_WITH_CLI=true \
  cargo install --git ssh://git@github.com/vickeryj/quorum-dispatch-clone.git \
    quorum-dispatch quorum-qw --rev <sha> --locked
# (once releases are tagged: --tag vX instead of --rev <sha>)
```

**Why the second selector.** `qd` delegates session work to a `qw` binary it spawns over stdio, and
it finds `qw` as a **sibling of its own executable** — never by searching `PATH`, because a `qw` on
`PATH` could belong to a different install than the running `qd`, which is precisely the version
skew the wire handshake exists to catch (ADR-0020). Installing `quorum-dispatch` alone therefore
produces a `qd` that cannot open a lane at all; it fails loudly, naming the path it looked for, and
never falls back. Both selectors land their binaries in the same `~/.cargo/bin`, from the same
`--rev`, which is exactly the invariant: **one directory, one install, one pin.** `qw` is not a
second pin and not a second thing to choose — it is a `[[bin]]` of an in-repo crate, and it has no
user-facing verbs.

Two non-obvious requirements (both verified the hard way):
- **`ssh://git@github.com/…`**, not scp-style `git@github.com:…` — cargo's URL parser rejects the
  latter (`relative URL without a base`).
- **`CARGO_NET_GIT_FETCH_WITH_CLI=true`** — the repos are private; cargo's built-in libgit2 fetch
  doesn't use your ssh-agent, so it must shell out to the git CLI to authenticate. Without it you get
  `failed to authenticate when downloading repository`.
- The `qd` package selector is required because `qd-rust` is a virtual workspace.

The binaries land at `~/.cargo/bin/qd` and `~/.cargo/bin/qw` — make sure that directory is on your
`PATH`. You only ever type `qd`; `qw` must be *present*, not reachable by name.

## 2. Bootstrap
```bash
qd bootstrap
```
`bootstrap` (idempotent, non-interactive-safe):
- creates `~/.quorum/dispatch` (+ `state/`),
- **re-points the relay MCP registration at this `qd` binary** (`current_exe`) — so moving/upgrading
  `qd` never again orphans the `~/.claude.json` relay path,
- offers shell integration (`eval "$(qd init bash)"`),
- on a TTY, offers to install the **pinned** `qb` binary and the **pinned** work-model plugin.

Extension install is currently **opt-in**: `QD_BOOTSTRAP_INSTALL_EXTENSIONS=1 qd bootstrap` (default off so
a plain bootstrap never reaches the network unasked — see ADR 0018 / tbd 0002 for the default-on question).

## 3. What gets installed, and from where
- **`qb`** binary ← `git@github.com:vickeryj/qb.git` @ the pinned rev (`cargo install --git … --bin qb`).
- **work-model plugin** ← `git@github.com:vickeryj/plugins.git` @ the pinned rev, installed via
  `claude plugin marketplace add … && claude plugin install core@qb`. `plugins/core` is consumed **raw**
  (no build step). The marketplace name (`qb`), plugin (`core`), and version (`0.1.0`) are held stable —
  commissions resolve roles by the cache path `~/.claude/plugins/cache/qb/core/0.1.0/roles/…`, so changing
  any of those three would move that path.

The actual install actions live in [`../scripts/install-extensions.sh`](../scripts/install-extensions.sh)
(outside `crates/`, so the engine stays content-free per `scope-audit.sh`). `qd bootstrap` invokes it by
path; a packaged install must stage that script and point `QD_INSTALL_EXTENSIONS_SCRIPT` at it.

## 4. Validate a pin before tagging (maintainers)
```bash
bash scripts/validate-pins.sh
```
Clones/fetches the pinned `qb` + `plugins` refs and confirms they exist and build. This is an **honest,
manual** gate — `extensions.toml` only bakes a string; there is no build-time validation in the binary.

## Self-update
`qd update` detects its install channel (Homebrew or cargo) and re-installs. (Homebrew + prebuilt
GitHub-Releases binaries for macOS/Linux are **v2** — see ADR 0018.)
