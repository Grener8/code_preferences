# Add Conditional Pause Control to LongRunningFunctionTool

## Objective

Enable **post-execution conditional pausing** for `LongRunningFunctionTool`, allowing tools to decide whether to pause **based on their return value**, instead of relying on a static `is_long_running=True`.

This is required to support:

* validation flows
* retry loops
* human-in-the-loop (HITL) confirmation only when necessary

---

## Problem Summary

### Current Behavior

* `LongRunningFunctionTool.is_long_running` is evaluated **before execution**
* If `True`, the system:

  * assigns `long_running_tool_ids`
  * pauses execution after tool call

### Resulting Issue

For tools with validation:

```text
LLM → tool(invalid input)
→ tool returns error
→ system pauses anyway
→ LLM never sees error
→ retry loop is broken
```

---

## Root Cause

Pause decision occurs in:

```text
_finalize_model_response_event → get_long_running_function_calls
```

This happens:

* before `handle_function_calls_async`
* before tool result is available

---

## Required Behavior

Pause must be determined using:

```text
function_call_result → decide pause
```

NOT:

```text
tool metadata → decide pause
```

---

## Proposed API

### Extend LongRunningFunctionTool

```python
LongRunningFunctionTool(
    func=submit_order,
    is_long_running: bool | Callable[[Any], bool]
)
```

---

### Behavior

#### Case 1 — Boolean (backward compatible)

```python
is_long_running=True
```

→ always pause (current behavior)

---

#### Case 2 — Callable

```python
def should_pause(result: dict) -> bool:
    return result.get("status") == "pending_confirmation"
```

* Tool executes first
* Result passed to `should_pause`
* Pause applied only if `True`

---

## Execution Flow (New)

### Current (Broken)

```text
LLM response
→ detect long-running tool
→ mark pause
→ execute tool
→ pause regardless of result
```

---

### Required Flow

```text
LLM response
→ execute tool
→ evaluate should_pause(result)
→ IF True:
     mark long_running_tool_ids
     pause
  ELSE:
     return result to LLM immediately
```

---

## Implementation Requirements

---

### 1. Defer Pause Decision

Move pause logic **after tool execution**.

* Do NOT assign `long_running_tool_ids` in `_finalize_model_response_event`
* Instead:

  * temporarily track candidate tool calls
  * finalize pause decision after execution

---

### 2. Modify Tool Execution Pipeline

In `handle_function_calls_async`:

```python
result = await tool.func(...)
```

Then:

```python
if callable(tool.is_long_running):
    should_pause = tool.is_long_running(result)
else:
    should_pause = tool.is_long_running
```

---

### 3. Apply Pause Conditionally

Only assign:

```python
event.long_running_tool_ids.add(call_id)
```

IF:

```python
should_pause is True
```

---

### 4. Ensure LLM Receives Non-Paused Results

If not paused:

* append `functionResponse` event
* continue LLM execution loop

---

## Required Invariants

```text
1. Tool result must always be visible to LLM unless explicitly paused
2. Pause must only occur when should_pause(result) == True
3. Validation errors must never trigger pause
4. Existing behavior must remain unchanged for boolean is_long_running
```

---

## Edge Cases

### Multiple Tool Calls

* Evaluate each independently
* Pause if **any** tool requires pause

---

### Exception Handling

* Exceptions should NOT trigger pause
* Should propagate as normal tool errors

---

### Streaming Compatibility

* If streaming enabled:

  * do NOT emit `long_running_tool_ids` prematurely
  * only emit after pause decision

---

## Backward Compatibility

* Existing usage must continue to work:

```python
LongRunningFunctionTool(func=..., is_long_running=True)
```

* No breaking API changes

---

## Validation Criteria

### Functional

* Tool returns validation error:

  * LLM receives error
  * LLM retries

* Tool returns confirmation-required:

  * execution pauses
  * session becomes resumable

---

### Regression

* Existing long-running flows unchanged
* No impact on non-long-running tools

---

### Failure Conditions

The implementation is incorrect if:

* validation errors still trigger pause
* LLM does not receive tool result
* pause occurs before tool execution
* callable `is_long_running` is ignored

---

## Rejected Approaches

| Approach                 | Problem                          |
| ------------------------ | -------------------------------- |
| Client-side auto-resume  | adds latency, breaks abstraction |
| Separate validation tool | incomplete, unsafe               |
| Always pause             | breaks retry logic               |

