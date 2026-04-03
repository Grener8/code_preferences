#   Fix LongRunningFunctionTool Resume + Streaming ID Instability in Google ADK

## Objective

Fix two coupled bugs in the **LongRunningFunctionTool resumability flow** when streaming is enabled:

1. **Incorrect pause detection after resume**
2. **Unstable function call IDs across streaming events**

The system must correctly:

* resume execution after receiving a valid `functionResponse`
* maintain stable function call IDs across partial and final events

---

## Context

Relevant components:

* `base_llm_flow.py`
* `llm_agent.py`
* `_run_one_step_async`
* `_finalize_model_response_event`
* `InvocationContext`
* Event model (`Event`, `functionCall`, `functionResponse`)

---

## Problem Summary

### Bug 1 — Resume Logic Falsely Detects Active Pause

#### Current Behavior

Resume logic:

```python
if (
    invocation_context.is_resumable
    and events
    and len(events) > 1
    and (
        invocation_context.should_pause_invocation(events[-1])
        or invocation_context.should_pause_invocation(events[-2])
    )
):
    return
```

This incorrectly assumes:

* pause state is determined by last 1–2 events

#### Failure Case

After a `functionResponse` is added:

* `events[-2]` still contains `long_running_tool_ids`
* `should_pause_invocation(events[-2]) == True`
* execution exits early → LLM is never called

---

### Expected Behavior

Pause should only persist if:

```text
EXISTS paused_tool_id NOT IN function_response_ids
```

---

## Bug 2 — Streaming Causes Function Call ID Mismatch

### Current Behavior

* `_finalize_model_response_event` generates new IDs via:

  ```python
  populate_client_function_call_id()
  ```

* With streaming:

  * partial events → sent via SSE (NOT stored)
  * final event → stored in session

### Result

* partial event ID ≠ final event ID
* client uses partial ID
* backend lookup fails:

```text
ValueError: Function call event not found
```

---

## Expected Behavior

Function call IDs must be:

```text
STABLE across all chunks of the same model response
```

---

## Required Fixes

---

### Fix 1 — Correct Pause Resolution Logic

Replace event-local check with **global resolution check**.

### Required Invariant

```text
A pause is active IFF:
any long_running_tool_id has no matching functionResponse.id
```

---

### Implementation Requirements

* Collect all functionResponse IDs across all events
* Compare against all `long_running_tool_ids`
* Only pause if unresolved IDs exist

---

### Example Logic

```python
all_response_ids = {
    fr.id
    for evt in events
    for fr in evt.get_function_responses()
    if fr.id
}

for evt in events:
    paused_ids = evt.long_running_tool_ids or set()
    if paused_ids - all_response_ids:
        return  # still paused
```

---

### Fix 2 — Preserve Function Call IDs Across Streaming

### Required Invariant

```text
All partial + final events derived from the same LLM response
must share identical functionCall.id values
```

---

### Implementation Requirements

* Detect prior partial event in stream
* Copy function call IDs forward into final event
* Prevent regeneration of IDs if already present

---

### Constraints

* `populate_client_function_call_id` must:

  * NOT overwrite existing IDs
* ID assignment must happen:

  * once per logical LLM response
  * not per chunk

---

### Acceptable Strategy

* Maintain temporary mapping:

```python
previous_event.function_calls → current_event.function_calls
```

* Apply before ID generation step

---

## Validation Criteria

### Functional

* After sending `functionResponse`:

  * agent continues execution
  * LLM is invoked again

* Streaming:

  * IDs in partial events == IDs in final event
  * no lookup failures

---

### Regression

* All existing unit tests must pass
* Add tests for:

  * multi-step resumable flows
  * streaming + resumability combined
  * multiple long-running tool calls

---

### Failure Conditions

The fix is invalid if:

* resume still exits early
* ID mismatch still occurs
* partial events produce different IDs
* non-streaming behavior is broken

---

## Notes

* Client-side filtering of partial events is NOT a valid fix
* Fix must be server-side and deterministic
* Avoid reliance on positional event checks (`events[-2]` etc.)
