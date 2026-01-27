# Implementation Plan: Native Supervisor Pattern Migration

**Duration:** 10-12 Days
**Scope:** Migrate from agents-as-tools to native LangGraph supervisor pattern

---

## Phase 1: Foundation & Architecture (Days 1-4)

### New `__init__.py` Files to Create

```
src/__init__.py
src/agents/__init__.py
src/agents/orchestrator/__init__.py
src/smart_ investigator/__init__.py
src/smart_ investigator/foundation/__init__.py
src/smart_ investigator/foundation/agents/__init__.py
src/smart_ investigator/foundation/schemas/__init__.py
src/smart_ investigator/foundation/tools/__init__.py
```

### Create `requirements.txt`

```
langgraph>=0.2.0
langchain-core>=0.3.0
langchain-openai>=0.2.0
mlflow>=2.17.0
pydantic>=2.0.0
databricks-sdk>=0.30.0
httpx>=0.27.0
httpx-sse>=0.4.0
backoff>=2.2.0
more-itertools>=10.0.0
fastapi>=0.110.0
pyyaml>=6.0.0
```

### New File: `src/agents/orchestrator/state.py`

Defines simplified state model:

```python
from langgraph.graph import MessagesState
from langgraph.graph.message import add_messages
from pydantic import BaseModel
from typing import Annotated, Literal

class ContextFrame(BaseModel):
    """Simplified version of TaskStruct - just what's needed"""
    agent_name: str
    context: str  # What the parent was asking for

class SupervisorState(MessagesState):
    """Simplified state with context stack for arbitrary nesting"""
    messages: Annotated[list, add_messages]
    current_agent: str
    context_stack: list[ContextFrame]

AGENT_NAMES = Literal[
    "decline_letter",
    "smart_strategy",
    "agent_3",
    "agent_4",
    "agent_5",
    "hitl",
    "end"
]
```

### New File: `src/agents/orchestrator/agent_config.py`

Replaces YAML-based tool configuration:

```python
AGENT_ENDPOINTS = {
    "decline_letter": {
        "endpoint_name": "decline-letter-agent",
        "description": "Crafts standardized declined letters",
        "introduction": "I can help you craft a declined letter."
    },
    "smart_strategy": {
        "endpoint_name": "smart-strategy-agent",
        "description": "Finds claim IDs",
        "introduction": "I can help you find your claim ID."
    },
    # Add other agents as needed
}
```

### New File: `src/agents/orchestrator/router.py`

Router node with LLM classification (~50 lines):

```python
ROUTER_PROMPT = """You are a router for an insurance assistant...
Available agents: {agents}
Context: {context}
User message: {user_message}
Respond with just the agent name."""

def create_router_node(llm):
    async def router_node(state: SupervisorState) -> dict:
        # Handle new conversation (welcome message)
        # Handle continuing conversation (LLM-based routing)
        # Return {"current_agent": agent_name}
    return router_node
```

### New File: `src/agents/orchestrator/agent_nodes.py`

Agent wrapper nodes for Databricks calls:

```python
def create_agent_node(agent_name: str, endpoint_config: dict):
    async def agent_node(state: SupervisorState, writer) -> dict:
        # Stream thinking message
        # Call Databricks endpoint
        # Handle context_stack push/pop for nested calls
        # Return updated state
    return agent_node
```

### New File: `src/agents/orchestrator/supervisor_graph.py`

Main graph with supervisor pattern:

```python
from langgraph.graph import StateGraph, START, END

def create_supervisor_graph(llm, agent_endpoints: dict) -> StateGraph:
    graph = StateGraph(SupervisorState)

    # Add router node
    graph.add_node("router", create_router_node(llm))

    # Add agent nodes (direct endpoint calls, not tools)
    for name, endpoint in agent_endpoints.items():
        graph.add_node(name, create_agent_node(name, endpoint))

    # Add HITL node
    graph.add_node("hitl", hitl_node)

    # Routing logic
    graph.add_edge(START, "router")
    graph.add_conditional_edges("router", route_to_agent, {
        "decline_letter": "decline_letter",
        "smart_strategy": "smart_strategy",
        "hitl": "hitl",
        "end": END
    })

    # All agents return to router
    for name in agent_endpoints:
        graph.add_edge(name, "router")
    graph.add_edge("hitl", "router")

    return graph
```

---

## Phase 2: Migrate Agents (Days 5-7)

### Modify: `src/agents/orchestrator/master_agent.py`

Switch from tools to supervisor graph:

```python
# Before
from agents.orchestrator.master_agent_graph import get_master_agent_graph
from agents.orchestrator.tools_set import tools as agent_tools

class MasterAgentResponsesAgent(LanggraphResponsesAgent):
    def get_graph(self) -> StateGraph:
        return get_master_agent_graph(self.llm, tools=agent_tools + [human_in_the_loop])

# After
from agents.orchestrator.supervisor_graph import create_supervisor_graph
from agents.orchestrator.agent_config import AGENT_ENDPOINTS

class MasterAgentResponsesAgent(LanggraphResponsesAgent):
    def get_graph(self) -> StateGraph:
        return create_supervisor_graph(self.llm, AGENT_ENDPOINTS)
```

### Modify: `src/smart_ investigator/foundation/tools/human_in_the_loop.py`

Simplify to work as a node instead of a tool:

```python
async def hitl_node(state: SupervisorState):
    """Human-in-the-loop node using LangGraph interrupt"""
    # Extract output to show user
    # Call interrupt() to pause for user input
    # Return updated state with user response
```

### Modify: `src/smart_ investigator/foundation/agents/response_agent.py`

Fix exception handling (replace bare `except:` clauses):

```python
# Before (lines 204, 218, 247, 255)
except:
    return []

# After
except Exception as e:
    logger.error(f"Error processing response: {e}")
    return []
```

---

## Phase 3: Update Streaming (Days 8-9)

### Event Compatibility Matrix

| Event | Current Source | New Source | Changes Needed |
|-------|---------------|------------|----------------|
| `ResponseTextDeltaEvent` | Tool streaming | Agent node streaming | Verify format |
| `ResponseOutputItemDoneEvent(thinking)` | `prepare_thinking_message` | Router node | Same format |
| `ResponseOutputItemDoneEvent(response)` | ToolMessage handling | Agent node response | Verify format |
| `ResponseCompletedEvent` | predict_stream | predict_stream | No change |

### Verify in `agent_nodes.py`

```python
# Ensure streaming events match frontend expectations
writer({
    "type": "response.output_text.delta",
    "delta": chunk_text,
    "item_id": item_id
})

# Ensure done events match
writer({
    "type": "response.output_item.done",
    "item": {
        "type": "message",
        "content": [...],
        "custom_outputs": {"event": "thinking", ...}
    }
})
```

### Test Streaming End-to-End

1. Start conversation - verify welcome message streams
2. Route to agent - verify thinking message appears
3. Agent responds - verify response streams correctly
4. HITL interrupt - verify interrupt event format
5. Resume - verify state preserved

---

## Phase 4: Cleanup (Days 10-11)

### Files to Delete

```
src/smart_ investigator/foundation/tools/tool_factory.py
src/smart_ investigator/foundation/tools/tool_set.py
src/smart_ investigator/foundation/tools/tool_set_utils.py
src/agents/orchestrator/tools_set.py
src/agents/orchestrator/master_agent_graph.py
```

### Simplify: `src/smart_ investigator/foundation/schemas/schemas.py`

Remove these schemas (no longer needed):
- `ToolMetadata`
- `ToolContentStruct`
- `ToolStruct`
- `ToolArgument`
- `TaskStruct`

Keep:
- `SupervisorState` (or move to state.py)
- `ContextFrame`
- Event types and error codes
- Frontend I/O types

### Simplify: `src/agents/orchestrator/master_agent_utils.py`

Remove:
- `_traceback_direct_request()`
- `format_task_struct()`
- Task stack management functions
- `IntentClassifierStruct`

Keep:
- HITL utility functions
- Message creation helpers
- Error handling utilities

### Delete Commented Code

Remove all commented-out code blocks across files. Git history preserves everything.

---

## Verification Checklist

### After Phase 1 (Day 4)
- [ ] All Python files compile without syntax errors
- [ ] New graph compiles: `python -c "from src.agents.orchestrator.supervisor_graph import create_supervisor_graph"`
- [ ] Router unit test passes with mock LLM

### After Phase 2 (Day 7)
- [ ] New conversation shows welcome message
- [ ] "craft a declined letter" routes to decline_letter agent
- [ ] HITL interrupt/resume works

### After Phase 3 (Day 9)
- [ ] Streaming events match before/after migration
- [ ] Frontend displays messages correctly

### After Phase 4 (Day 11)
- [ ] Full conversation flow works:
  ```
  User: I want to craft a declined letter
  → Routes to DeclineLetter
  User: Where to find a claim id?
  → Routes to SmartStrategy (nested call)
  User: I am a partner
  → SmartStrategy completes, returns to DeclineLetter
  User: My id is 123ABH
  → DeclineLetter continues
  User: Motor
  → Letter generated, conversation ends
  ```
- [ ] Error handling test: Databricks timeout returns error message
- [ ] No deleted files are imported anywhere

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
- `custom_outputs.master_agent_context` - Can be preserved if needed
- Streaming events - Same types, same format

### What Stays the Same
- MLflow ResponseAgent integration
- Databricks endpoint calls
- HITL interrupt/resume pattern
- All 5 sub-agents (just called differently)
- `predict_stream()` interface

### Risk Assessment
- **Medium risk** - Architecture change, but simpler result
- Nested calls handled via `context_stack` instead of `task_stack`
- Phase 3 has 2 days buffer for streaming format issues