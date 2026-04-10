# Batch Worker Briefing — v12 Overhaul

This document is the shared briefing for the 26-unit parallel overhaul that adds comprehensive Cheat Engine API coverage and fixes critical bugs in the MCP bridge. Every parallel worker reads this file; each worker is assigned exactly one unit and must follow the conventions below.

## Read first
1. `CLAUDE.md` at repo root — architecture briefing, commands, the 3-tier flow.
2. `MCP_Server/mcp_cheatengine.py` — Python FastMCP server. **The Windows stdio monkey-patch at lines 12–75 is load-bearing; do not touch unless your unit says so.**
3. `MCP_Server/ce_mcp_bridge.lua` — the Lua side, ~2316 lines. Landmarks: `toHex()` at 33-49, `serverState` at 16-27, `cleanupZombieState()` at 136-183, pure-Lua JSON codec at 186-289, `cmd_*` handlers at ~295-2055, `commandHandlers` dispatcher at ~2061, `PipeWorker` at ~2169.
4. `AI_Context/CE_LUA_Documentation.md` — authoritative CE 7.6 Lua API reference (~229 KB). Search for function names to find exact signatures.

**Language:** CE 7.6 ships Lua 5.3 with native int64 integers (verified in `AI_Context/plugins/lua.h` line 21 `LUA_VERSION_NUM 503` and `luaconf.h` which sets `LUA_INTEGER = __int64` on Windows). Use `//` for integer division, `&` `|` `~` for bitwise ops, and `string.format("0x%X", addr)` for 64-bit hex — no precision concerns.

**Concurrency invariant:** all CE Lua API calls MUST run on the main thread. The dispatcher handles that automatically — your `cmd_*` function is already on the main thread when invoked. Never spawn new threads that touch CE APIs directly.

## Section markers — enables clean parallel merges
Every handler you add MUST be wrapped so 25 sibling workers don't conflict with each other:

```lua
-- >>> BEGIN UNIT-NN <Title> <<<
local function cmd_xxx(params)
    -- ...
end
local function cmd_yyy(params)
    -- ...
end
-- >>> END UNIT-NN <<<
```

```python
# >>> BEGIN UNIT-NN <Title> <<<
@mcp.tool()
def xxx(...) -> str: ...
# >>> END UNIT-NN <<<
```

**Lua handler placement:** append your entire handler block immediately BEFORE the line `-- COMMAND DISPATCHER` (currently ~line 2057 of `ce_mcp_bridge.lua`). Do NOT append below the dispatcher — the `commandHandlers` table references your functions by name, so they must be defined first.

**Dispatcher placement:** add your dispatcher sub-block inside the `commandHandlers = { ... }` table, marker-wrapped, immediately before the closing `}`:

```lua
local commandHandlers = {
    -- existing entries...

    -- >>> BEGIN UNIT-NN dispatcher entries <<<
    xxx = cmd_xxx,
    yyy = cmd_yyy,
    -- >>> END UNIT-NN <<<
}
```

**Python placement:** append your `@mcp.tool()` block at the end of `mcp_cheatengine.py`, immediately BEFORE `if __name__ == "__main__":`.

## Return format
**Success:**
```lua
return { success = true, <domain fields> }
```

**Error:**
```lua
return { success = false, error = "<human-readable>", error_code = "<UPPER_SNAKE>" }
```

**Error code enum** (use only these):
`NO_PROCESS`, `INVALID_ADDRESS`, `INVALID_PARAMS`, `CE_API_UNAVAILABLE`, `DBVM_NOT_LOADED`, `DBK_NOT_LOADED`, `PERMISSION_DENIED`, `NOT_FOUND`, `OUT_OF_RESOURCES`, `INTERNAL_ERROR`.

## Address handling
**Output:** always hex string via `toHex(num)` (e.g. `"0x140001000"`).
**Input:** accept string or integer. Standard idiom (already at lines 1128, 1185, 1783 of the existing file):
```lua
if type(addr) == "string" then addr = getAddressSafe(addr) end
if not addr or addr == 0 then
    return { success = false, error = "Invalid address", error_code = "INVALID_ADDRESS" }
end
```

## Naming
Python tool name == Lua dispatcher key == snake_case verb-first. Lua function is `cmd_<name>`.
Example: `cmd_allocate_memory` → dispatcher `allocate_memory = cmd_allocate_memory` → Python `def allocate_memory(...)`.

## Python docstring template
```python
@mcp.tool()
def tool_name(param1: str, param2: int = 10) -> str:
    """<One-line summary>.

    Args:
        param1: <description>.
        param2: <description>.

    Returns JSON with: success, <field1>, <field2>.
    """
    return format_result(ce_client.send_command("tool_name", {"param1": param1, "param2": param2}))
```

## Hard rules
- NEVER return raw CE userdata — convert to primitives (`toHex` for addresses; plain numbers for ints; strings, arrays, tables for the rest).
- NEVER return Lua `nil` for "no result" — use empty array or `{success=false, error_code="NOT_FOUND"}`.
- ALWAYS wrap CE API calls in `pcall`; propagate errors via `error_code`.
- Every new handler that touches the TARGET PROCESS must check:
  ```lua
  if (getOpenedProcessID() or 0) == 0 then
      return { success = false, error = "No process attached", error_code = "NO_PROCESS" }
  end
  ```
  **Exempt** from this guard: CE-host operations (file I/O, clipboard, GUI dialogs, scripting, screen/input, CE-internal state).
- Do NOT edit existing `cmd_*` handlers, existing Python `@mcp.tool()` functions, or the Windows stdio monkey-patch unless your unit is specifically scoped to modify them.
- **Self-contained**: do NOT depend on helpers from sibling units (especially Unit 5's helpers, which land as a post-hoc refactor). If you need a helper, inline it.

## Forbidden
- Running `python MCP_Server/test_mcp.py` — it will hang forever waiting for the Named Pipe (Cheat Engine not available in worker sandbox).
- Committing untracked junk: `.claude/`, `__pycache__/`, `*.pyc`, or files outside your unit's scope.
- Force-pushing. Push with regular `git push -u fork HEAD`.
- Editing `.git/*`, modifying git config, running `git rebase -i`.

## Pre-commit checks (MANDATORY — paste the raw output into your PR body)

From the worktree root:

```bash
# 1. Python compile + AST parse
python -m py_compile MCP_Server/mcp_cheatengine.py && echo "py_compile OK"
python -c "import ast; ast.parse(open('MCP_Server/mcp_cheatengine.py', encoding='utf-8').read()); print('Py AST OK')"

# 2. Lua brace/bracket/paren balance (no luac on Windows)
python -c "
src = open('MCP_Server/ce_mcp_bridge.lua', encoding='utf-8').read()
for o, c, n in [('{','}','brace'),('[',']','bracket'),('(',')','paren')]:
    if src.count(o) != src.count(c):
        import sys; print(f'UNBALANCED {n}: {src.count(o)} vs {src.count(c)}'); sys.exit(1)
print('Lua balance OK')
"

# 3. Cross-file consistency check
python - <<'PY'
import re, sys
lua = open('MCP_Server/ce_mcp_bridge.lua', encoding='utf-8').read()
py  = open('MCP_Server/mcp_cheatengine.py', encoding='utf-8').read()

cmd_fns = set(re.findall(r'local function cmd_(\w+)', lua))
m = re.search(r'local commandHandlers\s*=\s*\{(.+?)^\}', lua, re.DOTALL | re.MULTILINE)
body = m.group(1) if m else ''
handler_keys = set(re.findall(r'^\s*(\w+)\s*=\s*cmd_\w+', body, re.MULTILINE))
mapped = set(re.findall(r'=\s*cmd_(\w+)', body))
py_sendcmds = set(re.findall(r'ce_client\.send_command\(\s*"(\w+)"', py))

orphan = cmd_fns - mapped
if orphan: print('ORPHAN cmd_*:', orphan); sys.exit(1)
missing = py_sendcmds - handler_keys
if missing: print('Python send_command with no Lua handler:', missing); sys.exit(1)

lb = sorted(re.findall(r'>>> BEGIN UNIT-(\S+?) ', lua))
le = sorted(re.findall(r'>>> END UNIT-(\S+?) ', lua))
if lb != le: print('Unbalanced Lua markers:', lb, 'vs', le); sys.exit(1)
pb = sorted(re.findall(r'>>> BEGIN UNIT-(\S+?) ', py))
pe = sorted(re.findall(r'>>> END UNIT-(\S+?) ', py))
if pb != pe: print('Unbalanced Python markers:', pb, 'vs', pe); sys.exit(1)

print(f'OK: {len(cmd_fns)} cmd_*, {len(handler_keys)} dispatcher keys, {len(py_sendcmds)} Python tools')
PY
```

If any check fails, fix before committing.

## Git workflow

The worktree has two remotes:
- `origin` = `https://github.com/miscusi-peek/cheatengine-mcp-bridge` (upstream, **read-only for you**)
- `fork` = `https://github.com/lauralex/cheatengine-mcp-bridge` (where you push; gh authed as lauralex)

```bash
# Verify you're on a worktree-isolated branch
git status
git log --oneline -3

# Make your edits, then:
git add -A
git commit -m "feat(unit-NN): <title>"

# Push to the FORK (not origin — you have no write access to origin)
git push -u fork HEAD

# Open PR targeting upstream main, head on lauralex's fork
gh pr create \
  --repo miscusi-peek/cheatengine-mcp-bridge \
  --base main \
  --head lauralex:<your-branch-name> \
  --title "feat(unit-NN): <title>" \
  --body "$(cat <<'EOF'
## Summary
<what this unit added/fixed>

## Unit
Unit NN — <Title>

## Tools added / changes
- <list>

## Static check output
\`\`\`
<paste the raw output from the pre-commit check commands above>
\`\`\`

## Testing
Static checks only. Live e2e validation deferred to user post-merge (worker sandbox has no Cheat Engine instance).
EOF
)"
```

If `gh pr create` against upstream fails (e.g., miscusi-peek doesn't accept PRs from lauralex), fall back to:
```bash
gh pr create --repo lauralex/cheatengine-mcp-bridge --base main --head <your-branch-name> --title "..." --body "..."
```
which opens a PR against the fork instead. If even that fails, note the failure and end with `PR: none — <reason>`.

## Deliverables (in order)
1. **Implement** — edit only the files in your unit's scope, following all conventions.
2. **Simplify** — invoke `Skill` with `skill: "simplify"` to clean up recent changes.
3. **Pre-commit checks** — run the three checks above, fix any failures, re-run until all pass.
4. **Skip e2e** — you cannot run Cheat Engine or `test_mcp.py`.
5. **Commit, push, PR** — use the exact git workflow above. Include the raw static-check output in the PR body.
6. **Report** — end your very last message with exactly one line:
   - `PR: <url>` — if the PR was created successfully
   - `PR: none — <reason>` — if any step blocked PR creation
