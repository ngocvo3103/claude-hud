# Security Policy

## Supported Versions

Security fixes are applied to the latest release series only.

## Reporting a Vulnerability

Please report security issues to: jarrodwttsyt@gmail.com

Include a clear description, reproduction steps, and any relevant logs or screenshots.
We will acknowledge receipt within 5 business days and provide a timeline for a fix if applicable.

## Threat Model

Claude HUD is a read-only statusline plugin for Claude Code. It renders real-time
information about your session. The following documents what the plugin accesses,
why, and the security controls in place.

### Data Access

| Data Source | What Is Read | Why | Security Controls |
|---|---|---|---|
| **stdin (from Claude Code)** | Model name, context window tokens, transcript path, CWD | Core HUD rendering | JSON-parsed with try/catch; malformed input returns null |
| **Transcript JSONL** | Tool use/result blocks, TodoWrite calls, Task calls | Tool, agent, and todo displays | Read-only streaming; malformed lines skipped |
| **`~/.claude/settings.json`** | MCP server count, hooks count | Environment line display | Parsed with try/catch; failures return zero counts |
| **`~/.claude/.credentials.json`** | OAuth access token, subscription type | Usage API authentication (legacy fallback) | Token is never logged; expiration validated before use |
| **macOS Keychain** | OAuth credentials (Claude Code 2.x) | Usage API authentication | Accessed via `/usr/bin/security` (absolute path); 3s timeout; backoff on failure |
| **`~/.claude/plugins/claude-hud/config.json`** | User preferences | HUD layout and display options | Parsed with try/catch; invalid config falls back to defaults |
| **Environment variables** | `HTTPS_PROXY`, `NO_PROXY`, `DEBUG`, `COLUMNS`, `CLAUDE_CONFIG_DIR`, `ANTHROPIC_BASE_URL` | Proxy, debug, terminal, and config resolution | All validated before use; never executed |
| **Git CLI** | Branch name, dirty status, ahead/behind counts | Git status display | Uses `execFile` (no shell); 1s timeout; CWD-scoped |

### Network Access

| Endpoint | Protocol | Purpose | Controls |
|---|---|---|---|
| `https://api.anthropic.com/api/oauth/usage` | HTTPS (TLS) | Fetch usage quota | Bearer token auth; configurable timeout (default 15s); proxy support with CONNECT tunneling |

No other network connections are made. The plugin does **not** phone home, collect
telemetry, or contact any third-party services.

### What the Plugin Does NOT Do

- Does not write to any file outside `~/.claude/plugins/claude-hud/` (cache only)
- Does not read `~/.ssh`, `~/.aws`, `~/.config`, or any secrets outside Claude's own config
- Does not install cron jobs, systemd units, or modify shell profiles
- Does not use `eval()`, `exec()` on dynamic strings, or `Function()` constructors
- Does not contain obfuscated code, base64-encoded payloads, or hidden logic
- Does not exfiltrate data via DNS, HTTP callbacks, or any side channel
- Has zero production dependencies (only dev dependencies for build/test)

### External Command Execution

The `--extra-cmd` CLI flag allows users to specify a custom command whose JSON
output is displayed in the HUD. This uses `exec()` with a shell, but:

- The command string is sourced exclusively from the user's own CLI arguments
- Output is sanitized to strip ANSI escapes, control characters, and bidi overrides
- Execution has a 3-second timeout and 10KB output buffer limit
- Only the `label` field from valid JSON output is used

### Supply Chain Hardening

- **Zero production dependencies**: The plugin uses only Node.js built-in modules at runtime
- **GitHub Actions pinned to SHA**: All CI/CD workflow actions reference immutable commit SHAs
- **Lock file integrity**: `package-lock.json` resolves all packages to `https://registry.npmjs.org/`
- **Dependabot enabled**: Automated weekly dependency update monitoring
- **Strict TypeScript**: `"strict": true` with full type checking
