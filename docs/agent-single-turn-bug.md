# Agent Single-Turn Bug

**File:** `HalCode9000.ailang` — `Function.CC_RunAgentMode`  
**Status:** Documented + Fixed  
**Severity:** Critical — agent tasks that require tool use silently produce no useful output  

---

## Problem Description

When `HalCode9000.x --agent deepseek` is invoked by `cc_agent_ipc` to run a sub-agent task, the main entry point is `CC_RunAgentMode`. This function is **single-turn**: it calls `CC_RunTurn` exactly once, extracts the last assistant message, writes it to stdout, and exits.

```
CC_RunAgentMode(prompt)
  └── History.AppendUser(prompt)
  └── CC_RunTurn()       ← ONE call only
  └── extract last assistant message
  └── CC_WriteStdout(result)
```

### What CC_RunTurn returns

`CC_RunTurn` runs the streaming API call and processes tool calls for **that one API round**. When the model responds with tool calls (e.g. `Pgmem op=pickup`, `Read`, `Grep`), `CC_RunTurn` dispatches them and appends the results to history — but then **returns**. The model never sees those results. There is no follow-up API call.

### The result

The agent does one of two things:

1. **If the model responds with pure text on turn 1** — works fine, result written to stdout.
2. **If the model makes any tool calls** — those tools execute, results go into history, but then the process exits. The last message in history at that point is the assistant's `tool_use` block (not a text response), so `CC_RunAgentMode`'s backward scan finds a message with no `"content"` string, writes `""` to stdout, and the daemon parks an empty string in Pgmem.

### Downstream effect

`cc_agent_ipc` spawns `HalCode9000.x --agent deepseek`, reads its stdout, and parks that as the task result. An empty stdout = empty Pgmem entry = the orchestrating HAL instance gets back nothing.

The task does get marked `complete` via `task_end`, so the orchestrator doesn't hang — it just gets an empty result and has no idea work was done.

---

## Root Cause

`CC_RunTurn` is not a loop — it is a single API request/response/dispatch cycle. The **full agentic loop** lives in `CC_ChatLoop`, which calls `CC_RunTurn` in a `WhileLoop` that continues as long as `keep_going = 1` (i.e., as long as the model keeps requesting tools). `CC_RunAgentMode` bypasses this loop entirely.

### CC_ChatLoop (correct pattern)

```
WhileLoop EqualTo(CCRunState.running, 1) {
    ...
    keep_going = 0
    WhileLoop EqualTo(keep_going, 1) {     ← agentic inner loop
        keep_going = 0
        CC_RunTurn()
        // if tools were dispatched, CC_RunTurn sets keep_going = 1 internally
    }
    // wait for next user input
}
```

Actually `CC_RunTurn` itself controls `keep_going` via its return value / the `n_tools > 0` path — the inner tool-dispatch loop sets `keep_going = 1` and loops back to call `CC_RunTurn` again.

### CC_RunAgentMode (broken pattern)

```
CC_RunTurn()      ← called once, returns, never loops back
```

---

## Fix

Replace the single `CC_RunTurn()` call in `CC_RunAgentMode` with a `keep_going` loop that mirrors the structure in `CC_ChatLoop`. The loop terminates when the model stops requesting tools (same as the interactive case).

A safety cap of 25 iterations is enforced to prevent runaway agents (matches the "max 25 tool chains" rule in the system prompt).

### Before (broken)

```ailang
Function.CC_RunAgentMode {
    Input: prompt: Address
    Body: {
        CC_AgentLog("[worker] CC_RunAgentMode starting")
        CCRunState.auto_approve = 1
        ...
        UI.Init()
        History.AppendUser(prompt)
        CC_AgentLog("[worker] CC_RunTurn starting")
        CC_RunTurn()
        CC_AgentLog("[worker] CC_RunTurn complete")
        UI.Shutdown()
        ...
    }
}
```

### After (fixed)

```ailang
Function.CC_RunAgentMode {
    Input: prompt: Address
    Body: {
        CC_AgentLog("[worker] CC_RunAgentMode starting")
        CCRunState.auto_approve = 1
        ...
        UI.Init()
        History.AppendUser(prompt)

        agent_turn = 0
        agent_max_turns = 25
        agent_keep_going = 1
        WhileLoop And(EqualTo(agent_keep_going, 1), LessThan(agent_turn, agent_max_turns)) {
            agent_keep_going = 0
            CC_AgentLog(StringConcat("[worker] CC_RunTurn turn=", NumberToString(agent_turn)))
            CC_RunTurn()
            CC_AgentLog("[worker] CC_RunTurn complete")

            // If model requested tools, CC_RunTurn appended results to history.
            // Check if the last history entry is a tool_result block — if so, loop.
            msgs = History.GetMessagesArray()
            cnt = History.Count()
            IfCondition GreaterThan(cnt, 0) ThenBlock: {
                last_tag = JSON.ArrayGet(msgs, Subtract(cnt, 1))
                last_msg = JSON.AsObject(last_tag)
                IfCondition NotEqual(last_msg, 0) ThenBlock: {
                    last_role = JSON.GetString(last_msg, "role")
                    IfCondition NotEqual(last_role, 0) ThenBlock: {
                        IfCondition EqualTo(StringCompare(last_role, "user"), 0) ThenBlock: {
                            // Last message is a user/tool-result block → loop back
                            agent_keep_going = 1
                        }
                    }
                }
            }
            agent_turn = Add(agent_turn, 1)
        }

        IfCondition GreaterEqual(agent_turn, agent_max_turns) ThenBlock: {
            CC_AgentLog("[worker] WARNING: hit max turn cap (25)")
        }

        UI.Shutdown()
        ...
    }
}
```

---

## Why "last role == user" works as the loop condition

After `CC_RunTurn` dispatches tool calls, it calls `Backend.AppendToolResultsToHistory(results)`. Anthropic-style history encodes tool results as a `"user"` role message containing `tool_result` content blocks. So if the last history message is `role: "user"`, it means tool results were just appended and the model needs another round. If the last message is `role: "assistant"` with text content, the model is done.

---

## Additional Notes

- **`CCRunState.auto_approve = 1`** is already set before the loop, so tool approvals are automatic (no interactive prompt needed in headless mode).
- **The 25-turn cap** matches the system prompt rule ("Max 25 tool chains in a row, then ask user"). In agent mode there is no user to ask, so we just log a warning and stop — the partial result is still written to stdout and parked.
- **The `ConnectProxy` path**: `CC_RegisterToolsOnly` calls `IPCDispatch.ConnectProxy()` which connects to `@halcode/AgentProxy`. All tool calls inside the loop are forwarded through the parent's already-open daemon connections — no new daemon processes are spawned.
- **Empty-result guard**: The backward history scan for the last assistant message remains unchanged. If somehow no assistant text is found (e.g. the model only ever used tools and the last tool round produced an error), `result` defaults to `""`. This is acceptable — the caller (cc_agent_ipc daemon) will log it.
