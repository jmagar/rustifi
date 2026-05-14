# AGENTS.md — rustifi

Rust binary MCP server for UniFi network controllers. Read-only REST API bridge exposing 8 actions.

## Essential Commands

```bash
cargo check                              # type-check (must pass before any PR)
cargo test                               # run all tests (no network required)
cargo run --bin unifi -- --help          # CLI help
cargo run --bin unifi -- health --json   # test a live action
cargo run --bin unifi                    # HTTP MCP server on :7474
cargo run --bin unifi -- mcp             # stdio MCP transport
```

## Architecture (strict layering)

```
UnifiClient (src/unifi.rs)   — HTTP only, no logic
    ↓
UnifiService (src/app.rs)    — all business logic
    ↓
tools.rs / cli.rs            — thin shims (parse + dispatch + format)
```

Never add business logic to tools.rs, cli.rs, or main.rs.

## Adding an Action

1. `src/unifi.rs` — add REST method
2. `src/app.rs` — delegate
3. `src/mcp/tools.rs` — match arm in dispatch()
4. `src/mcp/schemas.rs` — add to UNIFI_ACTIONS enum + schema
5. `src/mcp/rmcp_server.rs` — add to READ_ONLY_ACTIONS
6. `src/cli.rs` — CliCommand variant + parse + dispatch + formatter
7. Update HELP_TEXT in tools.rs and print_usage() in main.rs

## Key Files

- `src/unifi.rs` — UnifiClient, site_path() helper, UDM vs legacy path logic
- `src/config.rs` — UnifiConfig fields and env var names
- `src/mcp/schemas.rs` — JSON schema served to MCP clients
- `.env.example` — all supported env vars with documentation

## Environment

Minimum required:
```
UNIFI_URL=https://unifi.local
UNIFI_API_KEY=<your-api-key>
```

TLS: always set `UNIFI_SKIP_TLS_VERIFY=true` for self-signed controller certs.

## Tests

Tests in `tests/` do not require a live UniFi controller:
- `tool_dispatch.rs` — MCP dispatch (help, unknown action, missing action)
- `cli_parse.rs` — CLI argument parsing for all 8 commands

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->

## Plugin setup hooks

Plugin setup is owned by the binary. Keep `plugins/unifi/hooks/plugin-setup.sh` as a thin adapter that maps `CLAUDE_PLUGIN_OPTION_*` values to environment variables, prepares appdata, ensures `unifi` is on `PATH`, and then calls `unifi setup plugin-hook "$@"`.

`unifi setup check` is read-only, `unifi setup repair` is idempotent, and `unifi setup plugin-hook --no-repair` is audit mode. Do not add Docker Compose, systemd, or service bootstrap logic back into the hook script.
