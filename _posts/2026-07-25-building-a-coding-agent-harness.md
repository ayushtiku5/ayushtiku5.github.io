---
layout: post
title: "Building a Coding Agent Harness Directly on the Claude API"
date: 2026-07-25 11:00:00 -0000
categories: ai
description: "A hand-built agentic loop on the raw Claude API, with explicit tool execution, permission gating, and session persistence."
---

Claude Code, the Claude Agent SDK, and the API's own beta Tool Runner all give you an agentic loop already built. I wanted to build the loop by hand instead, on the raw Messages API, specifically so every step, what tool gets executed, what gets confirmed with the user first, what gets logged, is explicit code I own rather than something happening inside someone else's abstraction. Code is here: [github.com/ayushtiku5/agent-harness](https://github.com/ayushtiku5/agent-harness).

It's a CLI REPL: point it at a project directory, type a task, and it works the way Claude Code does, reading files, running shell commands, editing code, asking before anything risky.

## The loop itself

The whole agentic loop is about twenty lines. Send messages, and if Claude asks to use a tool, run it, append the result, and call again:

```python
def chat(self, user_input: str) -> None:
    self.messages.append({"role": "user", "content": user_input})

    for _ in range(MAX_TOOL_TURNS):
        response = self._request()
        if response is None:
            return
        if response.stop_reason == "refusal":
            print("[Claude declined this request.]")
            return
        if response.stop_reason == "pause_turn":
            self.messages.append({"role": "assistant", "content": response.content})
            continue
        if response.stop_reason != "tool_use":
            self.messages.append({"role": "assistant", "content": response.content})
            return

        self.messages.append({"role": "assistant", "content": response.content})
        tool_results = [self._execute(block) for block in response.content if block.type == "tool_use"]
        self.messages.append({"role": "user", "content": tool_results})
```

`MAX_TOOL_TURNS` caps it at 30 so a bad prompt or a confused model can't loop forever. `pause_turn` gets special handling: it means Claude's own turn was interrupted server-side (a long tool-use chain, for instance) and needs another request to continue, not a fresh user turn, so it loops without appending new user content. Everything else that isn't `tool_use` ends the turn and hands control back to the REPL.

## Tools: two built into the API, two I had to define

`bash` and `str_replace_based_edit_tool` are Anthropic-defined tools (`bash_20250124`, `text_editor_20250728`), meaning the API already knows their schema, the harness only supplies the executor:

```python
TOOL_DEFINITION = {"type": "bash_20250124", "name": "bash"}

def run(tool_input, *, cwd, timeout_seconds):
    if tool_input.get("restart"):
        return "bash session restarted (no persistent state was kept)"
    result = subprocess.run(
        tool_input.get("command"), shell=True, cwd=cwd,
        capture_output=True, text=True, timeout=timeout_seconds,
    )
    ...
```

Each `bash` call is a fresh subprocess, not a persistent shell, so a `cd` in one call doesn't carry over to the next. Search (`glob_files`, `grep_files`) had no equivalent built-in tool, so those needed a full custom JSON schema description written by hand, unlike the two that came for free.

## The permission gate: read-only runs free, everything else stops and asks

This is the part that makes the harness usable without babysitting every single step, and the part I was most careful with, since getting it wrong in either direction is bad: too strict and every trivial `ls` interrupts you, too loose and the agent silently deletes something.

```python
_UNSAFE_BASH_CHARS = re.compile(r"[;&|`$><]")

_SAFE_BASH_PREFIXES = (
    "git status", "git log", "git diff", "git show",
    "ls", "pwd", "echo", "cat", "head", "tail", "wc", "grep",
)

def _is_safe_bash(command: str) -> bool:
    command = command.strip()
    if not command or _UNSAFE_BASH_CHARS.search(command):
        return False
    if command in _SAFE_BASH_EXACT:
        return True
    return any(command == p or command.startswith(p + " ") for p in _SAFE_BASH_PREFIXES)
```

Notice the ordering: the character check runs before the prefix check, and it's an unconditional bailout, not just a guard on one branch. `ls; rm -rf .` starts with `ls`, but the semicolon fails it immediately regardless of what comes after. `find` is deliberately left off the safe list entirely, even though it looks read-only, because `-delete` and `-exec` can mutate the filesystem without needing any of the blocked shell metacharacters at all. That's the kind of exception that only shows up if you actually sit down and enumerate what each allowed command can do, not just what it usually does.

Text-editor calls are read-only only when the sub-command is literally `view`. `create` and `str_replace` always stop and ask. Anything that isn't obviously safe gets a plain confirmation prompt:

```python
description = _describe(tool_name, tool_input)
answer = input(f"\n[confirm] {description}\nAllow? [y/N] ").strip().lower()
if answer not in ("y", "yes"):
    raise PermissionDenied(f"User declined: {description}")
```

A decline doesn't crash the loop, it comes back as a `tool_result` with `is_error: True`, so Claude sees the refusal as part of the conversation and can adjust, same as it would if a command actually failed.

## Path containment: every filesystem path is untrusted input

Every path a tool touches, whether from `text_editor` or `search`, gets resolved against the configured project root and rejected if it escapes:

```python
def resolve_within(raw_path: str, *, root: Path) -> Path:
    candidate = Path(raw_path)
    candidate = candidate.resolve() if candidate.is_absolute() else (root / candidate).resolve()
    if not (candidate == root or root in candidate.parents):
        raise PathEscapesRoot(f"path '{raw_path}' escapes the project root {root}")
    return candidate
```

`.resolve()` follows symlinks, which is what catches a `../../etc/passwd`-style escape or a symlink planted inside the project that points somewhere else. This logic started duplicated inside `text_editor.py`, then got factored out into its own module once `search.py` needed the identical check. Security-critical logic duplicated across two files is exactly the kind of thing that drifts out of sync when one copy gets patched and the other doesn't, so consolidating it mattered more here than a typical refactor would.

There's a real bug the project's own roadmap records from this exact area: `Config.project_root` wasn't resolved unconditionally at startup, so on macOS, where `/tmp` is a symlink to `/private/tmp`, a path could pass the containment check computed one way and fail it computed the other way, depending on which form of the path showed up. Fixed by making `Config.__post_init__` always call `.resolve()` on the root itself, not just on the paths being checked against it, so both sides of every comparison are canonical.

## Persisting sessions without breaking the next API call

Conversations save to SQLite so `--resume <session-id>` can pick a task back up later. The tricky part wasn't the storage, it was that a streamed response's content blocks carry fields the API only produces, never accepts:

```python
_CONTENT_BLOCK_KEYS = {
    "text": {"type", "text"},
    "tool_use": {"type", "id", "name", "input"},
    "thinking": {"type", "thinking", "signature"},
}

def _sanitize_block(block: dict) -> dict:
    keys = _CONTENT_BLOCK_KEYS.get(block.get("type"))
    return {k: v for k, v in block.items() if k in keys} if keys else block
```

A live `response.content` object passed straight back into another API call within the same process works fine, the SDK handles it. The failure only shows up after a round trip through JSON storage and back: a blanket `model_dump(mode="json")` includes response-only fields like a text block's `parsed_output`, and replaying that as request input on a resumed session gets rejected with `Extra inputs are not permitted`. The fix is this explicit per-block-type allowlist, applied once before anything gets written to SQLite, so what comes back out is guaranteed to be valid request input regardless of what the response object happened to carry.

## What's built, and what's deliberately not

Two phases in, working: streaming output, prompt caching on the system prompt (verified directly, first call showed `cache_creation=1935`, an identical second call showed `cache_read=1935`), SQLite session persistence, a JSON config file with CLI-flag overrides, and a typed fallback chain for API errors (`NotFoundError` → `RateLimitError` → `APIStatusError` → `APIConnectionError`) so a failed request prints a message and ends the turn instead of an unhandled traceback killing the REPL mid-conversation.

Not built yet, and it says so directly in the project's own roadmap rather than pretending otherwise: context compaction for long sessions, per-tool permission policies instead of one blanket y/n gate, parallel execution of independent tool calls within a single turn, and task budgets so the model can self-pace on long agentic work. All reasonable next layers, but the part worth getting right first was the loop, the tool boundary, and the permission gate, since everything else builds on top of those three staying correct.
