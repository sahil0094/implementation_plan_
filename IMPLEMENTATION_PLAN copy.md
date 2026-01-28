# Implementation Plan: Native Supervisor Pattern Migration

**Duration:** 10-12 Days
**Scope:** Migrate from agents-as-tools to native LangGraph supervisor pattern

---

## Architectural Changes

### Current Architecture: Agents-as-Tools with Task Stack

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND REQUEST                                │
│  {input: [{content: [{text, custom_inputs}]}], custom_inputs: {state, ...}} │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         predict_stream() Entry                               │
│  1. Validate custom_inputs                                                   │
│  2. Init checkpointer (StatelessMemorySaver)                                │
│  3. Compile graph                                                            │
│  4. Preprocess → HumanMessage or Command(resume=...)                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LANGGRAPH EXECUTION LOOP                             │
│  ┌─────────────────────┐    ┌─────────────────────┐                         │
│  │  intent_classifier  │───▶│     tool_node       │                         │
│  │     (~200+ lines)   │    │                     │                         │
│  │                     │    │  • ToolFactory      │                         │
│  │  • Check task_stack │    │  • Execute tool     │                         │
│  │  • LLM routing      │◀───│  • Stream events    │                         │
│  │  • Create AIMessage │    │  • Return ToolMsg   │                         │
│  └─────────────────────┘    └─────────────────────┘                         │
│           │                          │                                       │
│           │ (end_condition)          │ (sub-agent call)                     │
│           ▼                          ▼                                       │
│        ┌──────┐            ┌───────────────────┐                            │
│        │ END  │            │ Databricks Mosaic │                            │
│        └──────┘            │    Endpoint       │                            │
│                            └───────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
          ┌─────────────────┐                 ┌─────────────────┐
          │  GraphInterrupt │                 │  Normal Complete │
          │  (HITL pause)   │                 │                  │
          └─────────────────┘                 └─────────────────┘
                    │                                   │
                    └─────────────────┬─────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STREAMING TO FRONTEND                                │
│  1. ResponseTextDeltaEvent (LLM tokens)                                     │
│  2. ResponseOutputItemDoneEvent (thinking/llm/response)                     │
│  3. ResponseCompletedEvent (final state for resume)                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### New Architecture: Native Supervisor

```
┌─────────────────────────────────────────────────────────────────┐
│                         SIMPLIFIED GRAPH                         │
│                                                                  │
│     ┌──────────┐                                                │
│     │  START   │                                                │
│     └────┬─────┘                                                │
│          │                                                       │
│          ▼                                                       │
│     ┌──────────┐      ┌──────────────┐                          │
│     │  router  │─────▶│ decline_letter│                         │
│     │  (~50    │      └──────┬───────┘                          │
│     │  lines)  │             │                                   │
│     │          │◀────────────┘                                   │
│     │          │                                                 │
│     │          │      ┌──────────────┐                          │
│     │          │─────▶│smart_strategy │                         │
│     │          │      └──────┬───────┘                          │
│     │          │◀────────────┘                                   │
│     │          │                                                 │
│     │          │      ┌──────────────┐                          │
│     │          │─────▶│    hitl      │ (interrupt)              │
│     │          │      └──────┬───────┘                          │
│     │          │◀────────────┘                                   │
│     │          │                                                 │
│     │          │─────▶ END                                       │
│     └──────────┘                                                │
│                                                                  │
│  State: { messages, current_agent, context_stack }              │
│  context_stack: [ContextFrame(agent_name, context)]  ← 2 fields │
└─────────────────────────────────────────────────────────────────┘
```

---

### Side-by-Side Comparison

```
CURRENT (Agents-as-Tools)              NEW (Native Supervisor)
─────────────────────────              ─────────────────────────

intent_classifier (200 lines)    →     router (50 lines)
        │                                      │
        ▼                                      ▼
    ToolNode                             conditional_edges
        │                                      │
        ▼                                      ▼
   ToolFactory                           agent_nodes
        │                                      │
        ▼                                      ▼
   task_stack                            context_stack
   [TaskStruct                           [ContextFrame
     [ToolStruct                           (agent_name,
       [ToolArgument]]]                     context)]

~800 lines                         →    ~400 lines
8+ schema classes                  →    3 simple models
```

---

## Key Issues with Current Design

1. **The `task_stack` Anti-Pattern**
   - Manually pushes/pops items to track conversation depth
   - Reinvents what LangGraph state machine already handles
   - Brittle nested conditions for stack management

2. **Monolithic `intent_classifier` Node**
   - Does too much: routing, formatting, state cleanup, prompt generation
   - Violates Single Responsibility Principle

3. **Excessive Schema Wrapping**
   - Custom classes wrap standard LangChain types
   - Adds serialization/deserialization overhead

4. **Complex Tool Factory**
   - Dynamically generates tool functions with decorators
   - Hard to statically analyze, type-check, or unit test

---

## Functionality Parity Checklist

| Feature | Current | New |
|---------|---------|-----|
| Task nesting (A→B→C→B→A) | `task_stack: List[TaskStruct]` | `context_stack: List[ContextFrame]` |
| LLM intent classification | `hitl_handler` with prompt | Router node with same prompt |
| Direct request (`is_direct`) | Bypasses LLM | Router checks flag first |
| `trusted` flag | Skips LLM on resume | Router respects flag |
| State preservation | `parent_state` in TaskStruct | `parent_state` in ContextFrame |
| Artifact passing | `ToolContentStruct.artifact` | `artifact` in ContextFrame |
| Completion detection | `need_resume` / `is_interrupt` | Same flags in agent node |
| Retry with backoff | `_get_final_event` retry loop | Same logic in agent node |

---

## Phase 1: Foundation & Architecture (Days 1-4)

### Task 1.1: Create Package Structure
- Add `__init__.py` files to all directories under `src/`
- Create `requirements.txt` with all dependencies (langgraph, langchain-core, mlflow, pydantic, databricks-sdk, httpx, etc.)

### Task 1.2: Create New State Model (`state.py`)
- Define `ContextFrame` dataclass with fields: `agent_name`, `context`, `parent_state`, `artifact`, `task_id`
- Define `SupervisorState` extending `MessagesState` with: `messages`, `current_agent`, `context_stack`, `is_direct`, `target_agent`, `trusted`, `current_artifact`
- Define `AGENT_NAMES` literal type for type safety

### Task 1.3: Create Agent Configuration (`agent_config.py`)
- Define `AgentEndpointConfig` TypedDict with: `endpoint_name`, `description`, `introduction`, `expose_to_user`, `can_generate_task`, `use_checkpointer`
- Create `AGENT_ENDPOINTS` dictionary mapping agent names to their configs
- Implement `get_agents_introduction()` helper for router prompt

### Task 1.4: Create Router Node (`router.py`)
- Implement router that handles 4 cases:
  1. New conversation → send to HITL for welcome
  2. Direct request (`is_direct=True`) → bypass LLM, route directly
  3. Trusted resume (`trusted=True`) → bypass LLM, return to parent
  4. Normal request → use LLM with tool binding for classification
- Use existing `IC_PROMPT_TEMPLATE` format for LLM routing
- Implement `route_to_agent()` conditional edge function

### Task 1.5: Create Agent Nodes (`agent_nodes.py`)
- Implement `create_agent_node()` factory function that:
  - Streams thinking messages via `prepare_thinking_message()`
  - Restores parent state from `context_stack` if resuming
  - Calls Databricks endpoint with proper request format
  - Handles completion detection via `is_interrupt` flag
  - On complete: pops `context_stack`, returns to parent or HITL
  - On interrupt: pushes new `ContextFrame`, routes to next agent
- Implement `_call_endpoint_with_retry()` with same retry semantics as current `_get_final_event()`

### Task 1.6: Create Supervisor Graph (`supervisor_graph.py`)
- Implement `create_supervisor_graph()` that:
  - Adds router node
  - Adds agent nodes for each endpoint in `AGENT_ENDPOINTS`
  - Adds HITL node
  - Defines edges: START → router, router → agents (conditional), agents → router

---

## Phase 2: Migrate Agents (Days 5-7)

### Task 2.1: Update Master Agent (`master_agent.py`)
- Change `get_graph()` to call `create_supervisor_graph()` instead of `get_master_agent_graph()`
- Add `_build_tools_for_router()` method to create minimal tool definitions for LLM classification
- Remove imports of old `tools_set` and `master_agent_graph`

### Task 2.2: Convert HITL to Node (`human_in_the_loop.py`)
- Convert from `@tool` decorator to async node function
- Handle 3 message cases: welcome (no messages), repeat (no context), agent request (has context)
- Build interrupt payload with same `Content` and `ContentCustomOutput` format
- Extract `is_direct` and `target_agent` from user's custom_inputs on resume

### Task 2.3: Fix Exception Handling (`response_agent.py`)
- Replace all bare `except:` clauses with `except Exception as e:`
- Add logging for all caught exceptions
- Ensure errors are propagated with context, not silently swallowed

---

## Phase 3: Update Streaming (Days 8-9)

### Task 3.1: Verify Event Format Compatibility
- Compare streaming events before/after migration
- Ensure `ResponseTextDeltaEvent` format unchanged
- Ensure `ResponseOutputItemDoneEvent` has correct `custom_outputs` structure
- Ensure `ResponseCompletedEvent` includes state metadata

### Task 3.2: Add Streaming to Agent Nodes
- Ensure thinking messages stream during routing decisions
- Ensure LLM tokens stream from sub-agent responses
- Verify interrupt events have correct format for frontend

### Task 3.3: End-to-End Streaming Tests
- Test: New conversation → welcome streams correctly
- Test: Route to agent → thinking message appears
- Test: Agent responds → tokens stream correctly
- Test: HITL interrupt → interrupt event format correct
- Test: Resume → state preserved, conversation continues

---

## Phase 4: Cleanup (Days 10-11)

### Task 4.1: Delete Obsolete Files
- Delete `src/smart_ investigator/foundation/tools/tool_factory.py`
- Delete `src/smart_ investigator/foundation/tools/tool_set.py`
- Delete `src/smart_ investigator/foundation/tools/tool_set_utils.py`
- Delete `src/agents/orchestrator/tools_set.py`
- Delete `src/agents/orchestrator/master_agent_graph.py`

### Task 4.2: Simplify Schemas (`schemas.py`)
- Remove: `ToolMetadata`, `ToolContentStruct`, `ToolStruct`, `ToolArgument`, `TaskStruct`
- Keep: Event types, error codes, frontend I/O types
- Move `SupervisorState` and `ContextFrame` to `state.py` if not already there

### Task 4.3: Simplify Utils (`master_agent_utils.py`)
- Remove: `_traceback_direct_request()`, `format_task_struct()`, task stack helpers, `IntentClassifierStruct`
- Keep: HITL utility functions, message creation helpers, error handling utilities

### Task 4.4: Remove Commented Code
- Delete all commented-out code blocks across all files
- Git history preserves everything if needed later

---

## Verification Checklist

### After Phase 1 (Day 4)
- [ ] All Python files compile without syntax errors
- [ ] New graph compiles successfully
- [ ] Router unit test passes with mock LLM
- [ ] Direct request test: `is_direct=True` bypasses LLM
- [ ] Trusted flag test: `trusted=True` bypasses LLM on resume

### After Phase 2 (Day 7)
- [ ] New conversation shows welcome message
- [ ] "craft a declined letter" routes to decline_letter agent
- [ ] HITL interrupt/resume works
- [ ] State preservation: agent state restored after nested call
- [ ] Artifact passing: artifacts flow between agents

### After Phase 3 (Day 9)
- [ ] Streaming events match before/after migration
- [ ] Frontend displays messages correctly

### After Phase 4 (Day 11)
- [ ] Full conversation flow works (see test scenario below)
- [ ] Direct request test passes
- [ ] Error handling test: Databricks timeout returns proper error
- [ ] No deleted files are imported anywhere

### Full Conversation Test Scenario
```
User: I want to craft a declined letter
→ Routes to DeclineLetter

User: Where to find a claim id?
→ Routes to SmartStrategy (nested call)
→ DeclineLetter state pushed to context_stack

User: I am a partner
→ SmartStrategy completes
→ Pops context_stack, returns to DeclineLetter

User: My id is 123ABH
→ DeclineLetter continues with restored state

User: Motor
→ Letter generated, conversation ends
```

---

## Summary

| Metric | Before | After |
|--------|--------|-------|
| Orchestration code | ~800 lines | ~400 lines |
| Number of schemas | 8+ tool-related | 3 simple models |
| Main routing function | 221 lines | ~50 lines |
| Files in orchestrator/ | 4 | 6 (but simpler) |

### What Changes for Frontend
**Likely no changes needed**, but verify:
- `custom_outputs.agent_name` - Still present
- `custom_outputs.master_agent_context` - Preserved
- `custom_inputs.is_direct` - Still supported
- `custom_inputs.agent_name` - Still supported
- Streaming events - Same types, same format

### What Stays the Same
- MLflow ResponseAgent integration
- Databricks endpoint calls
- HITL interrupt/resume pattern
- All 5 sub-agents (just called differently)
- `predict_stream()` interface
- Direct request functionality
- State preservation for nested calls
- Artifact passing between agents

### Risk Assessment
- **Medium risk** - Architecture change, but simpler result
- Nested calls handled via `context_stack` instead of `task_stack`
- Phase 3 has 2 days buffer for streaming format issues
- All existing functionality explicitly preserved

### Key Design Decisions
1. **ContextFrame replaces TaskStruct** - Same fields, cleaner structure
2. **Router handles all routing logic** - Direct, trusted, and LLM-based
3. **Completion detected via `is_interrupt`** - Same as before
4. **State preserved in `parent_state`** - Same as before
5. **Artifacts flow via `artifact` field** - Same as before
