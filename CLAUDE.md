# CLAUDE.md

Unofficial Rust CLI (`copilot`) that talks to Copilot Money's GraphQL API at `https://app.copilot.money/api/graphql`. This working copy is a **fork** of `JaviSoto/copilot-money-cli` owned by `ThePrismSystem`.

## Project map

- `src/main.rs` — binary entrypoint (calls `cli::run`)
- `src/lib.rs` — module wiring
- `src/cli/` — clap definitions + command dispatch (`mod.rs` is the top-level Cli struct + `run_transactions`; per-domain files: `auth.rs`, `categories.rs`, `recurrings.rs`, `tags.rs`, `budgets.rs`, `render.rs`)
- `src/client.rs` — `CopilotClient` with `Http` and `Fixtures` modes; all GraphQL calls go through `graphql()`
- `src/ops.rs` — `include_str!` constants mapping each operation name to its `.graphql` document
- `src/types.rs` — newtype IDs (`TransactionId`, `CategoryId`, etc.) + enums
- `src/config.rs` — token/session paths and file perms (`0600`/`0700`)
- `src/schema_gen.rs` + `src/bin/schema_gen.rs` — `schema-gen` binary (generates `schema/schema.graphql` stub)
- `graphql/*.graphql` — operation documents (one per `operationName`)
- `schema/schema.graphql` — best-effort schema stub
- `tests/` — `cli.rs` (behavior), `cli_snapshots.rs` + `tests/snapshots/` (insta), `client_http.rs` (mock server), `config.rs`, `tests/fixtures/graphql/*.json` (per-operation fixtures)
- `tools/` — Python helpers: `get_token.py` (Playwright auth), `test_get_token.py`, `capture_graphql_ops.py`
- `scripts/` — `release.sh`, `generate-demos.sh`, `update-coverage.sh`, `setup-dev.sh`
- `.githooks/` — repo hooks; activated by `scripts/setup-dev.sh` setting `core.hooksPath=.githooks`. `pre-commit` (fmt/test/clippy/gitleaks) is tracked; `commit-msg` (commitizen-based Conventional Commits check) is tracked; `pre-push` (fork-safety — blocks pushes to upstream) is local-only (see below)
- `.github/workflows/` — `ci.yml` (fmt/test/clippy/gitleaks/coverage), `release.yml` (tag-driven)

Tech stack: Rust 2024 edition · clap (derive) · reqwest (blocking + rustls) · serde/serde_json · comfy-table · rpassword · insta (snapshots).

<important if="you need to run commands to build, test, lint, or generate code">

| Command | Purpose |
|---|---|
| `cargo fmt --all` | Format |
| `cargo fmt --all -- --check` | Format check (CI + pre-commit) |
| `cargo test` | Run all tests |
| `cargo clippy -- -D warnings` | Lint (CI + pre-commit) |
| `cargo llvm-cov --workspace --summary-only` | Coverage summary |
| `cargo build --release --locked --bin copilot` | Release build |
| `cargo run --bin schema-gen -- --out schema/schema.graphql` | Regenerate schema stub |
| `scripts/generate-demos.sh` | Regenerate `assets/demo.gif` from `demo/basic.tape` (needs `vhs`) |
| `scripts/update-coverage.sh` | Update coverage badge in README |
| `scripts/release.sh <version>` | Bump version, tag, push, publish to crates.io |
| `scripts/setup-dev.sh` | Wire `.githooks/` as `core.hooksPath` |
| `python3 tools/test_get_token.py` | Unit tests for the auth helper |

</important>

<important if="you are adding a new CLI command or extending an existing one">

The end-to-end flow for a new command is:

1. Add/update the operation document in `graphql/<OperationName>.graphql`
2. Add an `include_str!` constant in `src/ops.rs`
3. Add a client method in `src/client.rs` that calls `self.graphql("OperationName", ops::..., json!({...}))` and shapes the response
4. Add clap structs + `Subcommand` variants in `src/cli/mod.rs` (or the relevant per-domain file)
5. Add a fixture JSON at `tests/fixtures/graphql/<OperationName>.json` (matches `ClientMode::Fixtures`)
6. Add/extend a snapshot test in `tests/cli_snapshots.rs` — snapshots live under `tests/snapshots/`
7. Update the **Command reference** in `README.md`

</important>

<important if="you are implementing a write action (any CLI that mutates server state)">

Every write command must route through the `confirm_write` pattern in `src/cli/mod.rs`:

- Respect `--dry-run` (print the planned change, return `Ok(())`, no server call)
- Respect `--yes` (skip confirmation)
- In non-interactive stdin, **refuse** without `--yes` (`anyhow::bail!("refusing to write in non-interactive mode without --yes")`)
- Otherwise prompt via `rpassword::prompt_password("Proceed? Type 'yes' to confirm: ")`

Existing examples: `TransactionsCmd::Review/Unreview/SetCategory/...` in `src/cli/mod.rs`.

</important>

<important if="you are writing or modifying tests">

- Prefer fixture-backed tests over live API calls. Set `COPILOT_FIXTURES_DIR` (or pass `--fixtures-dir`) to route the client through `ClientMode::Fixtures`.
- Fixture files are named `<OperationName>.json` and contain the raw GraphQL response body (`{"data": {...}}`).
- Snapshot tests use `insta`; update with `cargo insta review` (or delete the snapshot and re-run).
- HTTP-layer tests (`tests/client_http.rs`) use a mock server to exercise `ClientMode::Http` auth/retry/error paths.
- The helper in `src/client.rs` reads `COPILOT_TEST_REFRESH_TOKEN` to stub session refresh deterministically.

</important>

<important if="you are touching auth, tokens, or session files">

- Bearer token at `~/.config/copilot-money-cli/token` (perms `0600`, set in `src/config.rs`).
- Playwright session at `~/.config/copilot-money-cli/playwright-session/` (perms `0700`). Created by `--persist-session`.
- Env overrides: `COPILOT_TOKEN`, `COPILOT_TOKEN_FILE`, `COPILOT_SESSION_DIR`, `COPILOT_BASE_URL`.
- Session refresh calls `python3 tools/get_token.py --mode session` (located via `config::token_helper_path` — source checkout OR `libexec/copilot-money-cli/get_token.py` next to the binary).
- Never log or print the token value itself. `copilot auth status` reports `token_configured`/`token_valid` only.
- `gitleaks` runs in CI and the pre-commit hook — do not commit anything that looks like a bearer/credential.

</important>

<important if="you are modifying the GraphQL surface (adding/changing queries, mutations, variables)">

- Each operation is a standalone `.graphql` file under `graphql/` and must be loaded via `include_str!` in `src/ops.rs` — the `operationName` sent to the server must match the file's operation name.
- The client posts to `{base_url}/api/graphql` with `{"operationName", "query", "variables"}` and attaches `Authorization: Bearer <token>`.
- Unauthenticated responses (`errors[0].extensions.code == "UNAUTHENTICATED"`) trigger one automatic session-based refresh + retry; other errors bubble up via `format_graphql_error`.
- `tools/capture_graphql_ops.py` records the live app's GraphQL traffic to `artifacts/graphql-ops/` — use it to discover new operations, then hand-write the `.graphql` file.

</important>

<important if="you are running git, opening a PR, or pushing commits">

**This clone is a fork — upstream is `JaviSoto/copilot-money-cli` and MUST NOT receive pushes or PRs from Claude.**

- Remotes: `origin` = upstream (read-only), `fork` = `ThePrismSystem/copilot-money-cli` (your destination).
- `git config remote.pushDefault fork` is set — plain `git push` goes to the fork.
- `.githooks/pre-push` (local-only, excluded via `.git/info/exclude`) refuses any push to `origin` or any JaviSoto URL.
- `gh repo set-default ThePrismSystem/copilot-money-cli` is set — prefer `gh pr create --repo ThePrismSystem/copilot-money-cli` explicitly.
- `.claude/settings.local.json` has a PreToolUse hook that blocks: `gh pr create` without `--repo ThePrismSystem/...`, and `git push --no-verify` to origin.
- PRs targeting the upstream are opened by the human via the GitHub web UI — never via CLI from this session.

</important>

<important if="you are cutting a release">

- Release is tag-driven (`v<version>`). The workflow at `.github/workflows/release.yml` builds Linux/macOS tarballs, uploads to GitHub Releases, and updates the `JaviSoto/homebrew-tap` Formula via `HOMEBREW_TAP_TOKEN`.
- Use `scripts/release.sh <version>` — it runs fmt/test/clippy, bumps `Cargo.toml`, adds a `CHANGELOG.md` entry, commits, tags, pushes both branch and tag, and runs `cargo publish`.
- The release must be driven from upstream (`JaviSoto/*`), not this fork — do not invoke `scripts/release.sh` from here without explicit user direction.

</important>

<important if="you are debugging the Python auth helper (tools/get_token.py)">

- Set `COPILOT_DEBUG_GET_TOKEN=1` to enable `trace()` output on stderr.
- The helper re-execs into `~/.codex/integrations/venv` if present (for the author's Gmail-magic-link integration). On systems without that venv, that branch is a no-op and the helper falls back to manual link paste.
- For headless Linux, the helper auto re-execs under `xvfb-run` when no `$DISPLAY`/`$WAYLAND_DISPLAY` is set and `xvfb-run` is available.
- The helper only writes to stdout the captured bearer token; all diagnostics go to stderr. The Rust parent captures stdout, trims, and writes to the token file.

</important>
