# Installing the sb stack (sb + sbx + plugins)

One version of `sb` determines the matched `sbx` + work-model plugin. You pin `sb`; `sb` carries the
rest in [`../extensions.toml`](../extensions.toml).

## Prerequisites
- A Rust toolchain (`cargo`).
- SSH access to the private `private-org` GitHub repos (cargo pulls over SSH).
- The `claude` CLI on `PATH` (for the work-model plugin install).

## 1. Install the engine (pinned)
`sb-rust` is a **virtual workspace**, so the install command needs a package selector:

```bash
CARGO_NET_GIT_FETCH_WITH_CLI=true \
  cargo install --git ssh://git@github.com/private-org/sb-rust.git sb --rev <sha> --locked
# (once releases are tagged: --tag vX instead of --rev <sha>)
```

Two non-obvious requirements (both verified the hard way):
- **`ssh://git@github.com/…`**, not scp-style `git@github.com:…` — cargo's URL parser rejects the
  latter (`relative URL without a base`).
- **`CARGO_NET_GIT_FETCH_WITH_CLI=true`** — the repos are private; cargo's built-in libgit2 fetch
  doesn't use your ssh-agent, so it must shell out to the git CLI to authenticate. Without it you get
  `failed to authenticate when downloading repository`.
- The `sb` package selector is required because `sb-rust` is a virtual workspace.

The binary lands at `~/.cargo/bin/sb` — make sure that's on your `PATH`.

## 2. Bootstrap
```bash
sb bootstrap
```
`bootstrap` (idempotent, non-interactive-safe):
- creates `~/.sb` (+ `state/`),
- **re-points the relay MCP registration at this `sb` binary** (`current_exe`) — so moving/upgrading
  `sb` never again orphans the `~/.claude.json` relay path,
- offers shell integration (`eval "$(sb init bash)"`),
- on a TTY, offers to install the **pinned** `sbx` binary and the **pinned** work-model plugin.

Extension install is currently **opt-in**: `SB_BOOTSTRAP_INSTALL_EXTENSIONS=1 sb bootstrap` (default off so
a plain bootstrap never reaches the network unasked — see ADR 0018 / tbd 0002 for the default-on question).

## 3. What gets installed, and from where
- **`sbx`** binary ← `git@github.com:private-org/sbx.git` @ the pinned rev (`cargo install --git … --bin sbx`).
- **work-model plugin** ← `git@github.com:private-org/plugins.git` @ the pinned rev, installed via
  `claude plugin marketplace add … && claude plugin install core@sbx`. `plugins/core` is consumed **raw**
  (no build step). The marketplace name (`sbx`), plugin (`core`), and version (`0.1.0`) are held stable —
  commissions resolve roles by the cache path `~/.claude/plugins/cache/sbx/core/0.1.0/roles/…`, so changing
  any of those three would move that path.

The actual install actions live in [`../scripts/install-extensions.sh`](../scripts/install-extensions.sh)
(outside `crates/`, so the engine stays content-free per `scope-audit.sh`). `sb bootstrap` invokes it by
path; a packaged install must stage that script and point `SB_INSTALL_EXTENSIONS_SCRIPT` at it.

## 4. Validate a pin before tagging (maintainers)
```bash
bash scripts/validate-pins.sh
```
Clones/fetches the pinned `sbx` + `plugins` refs and confirms they exist and build. This is an **honest,
manual** gate — `extensions.toml` only bakes a string; there is no build-time validation in the binary.

## Self-update
`sb update` detects its install channel (Homebrew or cargo) and re-installs. (Homebrew + prebuilt
GitHub-Releases binaries for macOS/Linux are **v2** — see ADR 0018.)
