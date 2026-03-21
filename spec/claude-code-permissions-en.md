# Claude Code Permission System Specification

Updated: 2026-03-21


## Overview

Claude Code prompts users for permission when executing tools.\
The frequency of these prompts can be controlled via `settings.json`.


## Permission Modes

| Mode | Behavior | Use case |
| --- | --- | --- |
| `default` | Prompts on first use of each tool | Normal development |
| `acceptEdits` | Auto-approves file edits; other tools still prompt | Trusted iteration |
| `plan` | Analysis only; no file changes or command execution | Code review, proposals |
| `bypassPermissions` | Skips prompts except for protected directories | Containers / isolated VMs only |


## Settings File Precedence

Higher entries take priority.

1. Managed settings (admin-controlled, cannot be overridden)
2. Command-line arguments (session-only)
3. `.claude/settings.local.json` (local project, not committed)
4. `.claude/settings.json` (shared project, committed)
5. `~/.claude/settings.json` (user global)


## Permission Rule Structure

Three rule types are available: `allow`, `ask`, and `deny`.

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm run *)",
      "Bash(git commit *)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```


### Rule Evaluation Order

1. **deny** — First match blocks immediately (overrides allow)
2. **ask** — Shows confirmation dialog
3. **allow** — Auto-approves


## Tool-Specific Configuration


### Bash Commands

Use wildcards (`*`) for pattern matching.

```json
"Bash(npm run *)"      // Matches npm run build, npm run test, etc.
"Bash(git add *)"      // Matches git add ., etc.
"Bash(git * main)"     // Matches git checkout main, git merge main, etc.
```

> **Word boundary matters**: `Bash(ls *)` matches `ls -la` but NOT `lsof`.\
> Without the space (`Bash(ls*)`), both match.

> **Compound commands**: Approving `git status && npm test` saves separate rules for each subcommand (up to 5).


### Read / Edit (Path Patterns)

Four path types following gitignore specification.

```json
"Read"                      // Allow reading all files
"Edit(/src/**/*.ts)"        // Relative to project root
"Read(~/Documents/*.pdf)"   // Relative to home directory
"Read(//etc/hosts)"         // Absolute filesystem path
```


### WebFetch

Control by domain.

```json
"WebFetch(domain:github.com)"
"WebFetch(domain:api.example.com)"
```


### MCP Server Tools

```json
"mcp__puppeteer"                       // All tools from server
"mcp__puppeteer__puppeteer_navigate"   // Specific tool
"mcp__slack__*"                        // Wildcard
```


## Operations That Should Always Require Confirmation

The following should be set to `deny` or `ask`, never `allow`.

- `Bash(git push *)` — Pushes changes to remote
- `Bash(git push --force *)` — Overwrites remote history
- `Bash(rm -rf *)` — Irreversible deletion
- `Bash(sudo *)` — Privileged operations
- `Edit(//.git/**)` — Git metadata modification
- `Edit(//.env)` — Secret file modification

> **Read/Edit deny does not block Bash.**\
> Denying `Read(//.env)` does not prevent `Bash(cat .env)`.\
> For sensitive files, also deny `Bash(cat .env)` or use OS-level sandboxing.


## Operations Safe to Auto-Allow

The following can be set to `allow` with minimal security impact.

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm run build)",
      "Bash(npm run test)",
      "Bash(npm run dev)",
      "Bash(npm run lint)",
      "Bash(npm run lint:fix)",
      "Bash(npm install)",
      "Bash(npx *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git checkout *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git branch *)",
      "Bash(node --version)",
      "Bash(npm --version)"
    ]
  }
}
```


## How to Reduce Permission Prompts

1. **Register frequent commands in `settings.json`.**\
Add project-specific build and test commands to `allow`.

2. **Use "Yes, don't ask again" during sessions.**\
This auto-approves similar operations for the rest of the session.

3. **Use `acceptEdits` mode.**\
After confirming the approach, skip file edit confirmations.

4. **Create a project-level `.claude/settings.json`.**\
Share team-agreed safe defaults.

5. **Combine `deny` with `WebFetch`.**\
Deny `Bash(curl *)` and use `WebFetch(domain:...)` with domain restrictions instead of controlling URLs via Bash patterns.


## Bash Pattern Limitations

Bash pattern matching has structural limitations.

- `curl http://github.com/*` does not match `curl -X GET http://github.com/...` (option before URL)
- Does not account for `https://` protocol or redirects
- Cannot detect variable expansion (`URL=http://... && curl $URL`)

For network access control, use `WebFetch` domain restrictions instead of Bash patterns.


## --dangerously-skip-permissions Flag

```bash
claude "your task" --dangerously-skip-permissions
```

Skips all permission prompts.\
Equivalent to `bypassPermissions` mode.

**Acceptable use cases:**

- CI/CD containers
- Docker development environments
- Disposable VMs

**Never use on:**

- Development machines with important files
- Shared systems
- Production servers


## Protected Directories

Even in `bypassPermissions` mode, writes to these directories still require confirmation.

- `.git/**`
- `.claude/**` (except `.claude/commands`, `.claude/agents`, `.claude/skills`)
- `.vscode/**`
- `.idea/**`


## /permissions Command

```
/permissions
```

Lists all active rules (allow / ask / deny).\
Shows which settings file each rule comes from.


## Difference Between CLAUDE.md and settings.json

| Aspect | CLAUDE.md | settings.json |
| --- | --- | --- |
| Role | Prompt instructions (behavioral guidance) | Tool execution permission control |
| Permission control | Not possible | Possible |
| Scope | Affects Claude's responses and decisions | Affects confirmation dialogs on tool calls |

Writing "you may edit this folder" in CLAUDE.md does NOT skip tool permission prompts.\
Pre-approval of permissions must be done via `permissions.allow` in `settings.json`.


## Project Configuration Example

The following is configured in `~/.claude/settings.json` to minimize confirmation prompts during development.

### allow (auto-approve)

**File operations:**

```json
"Read",
"Edit(/anytime-markdown/**)",
"Edit(/anytime-markdown-docs/**)",
"Edit(/home/node/.claude/**)",
"Write(/anytime-markdown/**)",
"Write(/anytime-markdown-docs/**)",
"Write(/home/node/.claude/**)"
```

> `Read` allows all files without path restrictions.\
> `Edit` / `Write` are limited to project directories and Claude Code config directory.

**Bash commands:**

```json
"Bash(npm *)",
"Bash(npx *)",
"Bash(git status *)",
"Bash(git diff *)",
"Bash(git log *)",
"Bash(git add *)",
"Bash(git commit *)",
"Bash(git checkout *)",
"Bash(git merge *)",
"Bash(git pull *)",
"Bash(git branch *)",
"Bash(docker *)",
"Bash(bash *)"
```

### ask (require confirmation)

```json
"Bash(git push *)",
"Bash(git push)"
```

> Pushing to remote always requires confirmation.

### deny (always block)

```json
"Bash(rm -rf *)",
"Bash(sudo *)"
```


### Migration Note: Old Format

The Bash pattern format has changed.

| Old format | New format |
| --- | --- |
| `Bash(git status:*)` | `Bash(git status *)` |
| `Bash(npm:*)` | `Bash(npm *)` |

The `:*` format is from older versions.\
Current versions use `Bash(command *)` with a space separator.


## Tip: Switching Permission Modes Mid-Session

Press `Shift+Tab` to switch permission modes instantly during a session.\
A practical workflow: confirm the approach first, then switch to `acceptEdits` to skip file edit confirmations during implementation.


## Best Practices (as of 2026)


### Choosing the Right Permission Mode

| Phase | Recommended Mode | Reason |
| --- | --- | --- |
| Initial / unfamiliar code | `default` | Confirm all operations, prevent unintended changes |
| Implementation after plan is set | `acceptEdits` | Skip file edit prompts, keep Bash confirmation |
| Code review / analysis | `plan` | Read-only, no modifications allowed |
| CI/CD / isolated environments | `bypassPermissions` | Sandboxed environments only |

> Always Git commit before switching to `acceptEdits`.\
> Unintended changes can be reverted with `git checkout`.


### Deny List Design (Blocklist-First)

The 2026 best practice is "define deny first, add allow only as needed".\
All users should configure a deny list, even in `default` mode.

**Operations to include in deny:**

- `Bash(rm -rf *)` — Irreversible deletion
- `Bash(sudo *)` — Privileged operations
- `Bash(git push --force *)` — History overwrite
- `Bash(chmod 777 *)` — Excessive permissions
- `Bash(curl *)` / `Bash(wget *)` — Arbitrary external communication (use WebFetch domain restrictions instead)

> For projects with database operations, also deny destructive SQL commands like `Bash(DROP TABLE *)`.


### PreToolUse Hooks (Advanced Control)

For cases where `settings.json` pattern matching is insufficient, use PreToolUse hooks.\
Shell scripts inspect tool calls before execution: exit code `0` to allow, `2` to block.

```bash
# Example: ~/.claude/hooks/pre-tool-use.sh
# Block dangerous patterns
if echo "$INPUT" | grep -q "rm -rf /"; then
  echo "Blocked: dangerous rm command" >&2
  exit 2
fi
exit 0
```

> Effective for compensating pattern matching limitations (variable expansion, option order differences, etc.).


### MCP Server Permission Management

- Review server source code and official documentation before installation
- MCP servers that fetch untrusted content carry prompt injection risk
- `enableAllProjectMcpServers: true` is convenient but enables unverified servers
- Explicitly approve only trusted servers


### Combining with Sandboxing

The permission system and sandboxing provide defense in depth.

| Layer | Role |
| --- | --- |
| Permission system | "Should this tool be executed?" |
| Sandbox | "What can it access when executed?" |

> Denying `Read(//.env)` does not prevent `Bash(cat .env)`.\
> Use OS-level sandboxing (macOS: Seatbelt, Linux: bubblewrap) as a complement.

**Sandbox settings to avoid:**

- `allowUnixSockets` — Risk of sandbox escape via Docker socket
- Write access to `$PATH` directories or `.bashrc` — Privilege escalation risk


### Enterprise / Team Environments

Use `managed-settings.json` for enforcing unified policies across teams.\
Individual users cannot override these settings.

| OS | Path |
| --- | --- |
| Linux / WSL | `/etc/claude-code/managed-settings.json` |
| Windows | `C:\ProgramData\ClaudeCode\managed-settings.json` |

Share project-specific settings via `.claude/settings.json` (committed), and manage personal overrides in `.claude/settings.local.json` (add to `.gitignore`).


### Implementation Checklist

- Destructive commands registered in deny list
- Git committed before switching to `acceptEdits`
- MCP servers reviewed and explicitly approved
- `.claude/settings.local.json` added to `.gitignore`
- `bypassPermissions` used only in sandboxed environments


## References

| Article | Content |
| --- | --- |
| [Claude Code settings.json: Complete config guide](https://www.eesel.ai/blog/settings-json-claude-code) | Comprehensive settings guide with wildcard pattern examples |
| [A complete guide to Claude Code permissions](https://www.eesel.ai/blog/claude-code-permissions) | Rule evaluation order, settings hierarchy, security considerations |
| [YOLO Mode: Permission configuration](https://dev.to/rajeshroyal/yolo-mode-when-youre-tired-of-claude-asking-permission-for-everything-2daf) | `Shift+Tab` mode switching, practical `acceptEdits` usage |
| [Claude Code Auto Approve guide](https://smartscope.blog/en/generative-ai/claude/claude-code-auto-permission-guide/) | `acceptEdits` mode in daily development workflows |
| [Claude Code Security Best Practices](https://www.backslash.security/blog/claude-code-security-best-practices) | Recommended allow/deny list design, MCP server security |
| [Better Claude Code permissions](https://blog.korny.info/2025/10/10/better-claude-code-permissions) | "Allow only routine, low-risk operations" design philosophy |
| [dangerously-skip-permissions: Safe Usage Guide](https://www.ksred.com/claude-code-dangerously-skip-permissions-when-to-use-it-and-when-you-absolutely-shouldnt/) | Safe use cases and caveats for bypassPermissions |
| [How I use Claude Code (+ my best tips)](https://www.builder.io/blog/claude-code) | Practical permission setup from a development team's daily workflow |
| [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) | Official best practices from Anthropic |
| [Hooks reference](https://code.claude.com/docs/en/hooks) | Official PreToolUse hooks reference |
| [Sandboxing](https://code.claude.com/docs/en/sandboxing) | Official sandboxing documentation |
| [Enterprise configuration](https://support.claude.com/en/articles/12622667-enterprise-configuration) | Unified team management with managed-settings.json |
| [Secure Your Claude Skills with Custom PreToolUse Hooks](https://egghead.io/secure-your-claude-skills-with-custom-pre-tool-use-hooks~dhqko) | PreToolUse hooks implementation tutorial |
| [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config) | Strict security configuration template by Trail of Bits |
