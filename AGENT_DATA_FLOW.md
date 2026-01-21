# Agent Data Flow: What Gets Passed Within and Across Agents

## Overview

This document describes exactly what data items are passed to agents, what they build internally, what they return, and how data accumulates across agent executions and conversation turns.

---

## 1. What Agents Receive (INPUT from GraphState)

When an agent executes, it receives the full `GraphState` object containing:

### Core Conversation Data

- **`messages`**: `List[BaseMessage]` 
  - All historical messages from ALL previous agents/turns
  - Accumulated via `operator.add` (never cleared during session)
  - Includes messages from all agents across all conversation history

- **`current_message`**: `BaseMessage`
  - Current user message (separated from history)

### Routing Context

- **`router_decision`**: `RouterDecision`
  - `primary_agent`: Which agent was selected
  - `complexity`: Simple or complex query
  - `time_ranges`: Extracted time periods for memory/data retrieval
  - `secondary_topics`: Other topics detected

- **`routing_history`**: `List[RouterDecision]`
  - Last N routing decisions

### Previous Agent Outputs (Same Turn)

- **`agent_outputs`**: `Dict[str, AgentOutput]`
  - Outputs from agents that already executed in this turn
  - Each `AgentOutput` contains:
    - `agent_name`: Name of the agent
    - `content`: Text response
    - `structured_data`: Tool results (JSON) for downstream agents
    - `tools_called`: List of tool names
    - `tool_timestamps`: Execution timing
    - `new_messages`: AIMessages + ToolMessages generated

### User/Session Context

- **`auth_ctx`**: `AuthContext` - User ID, permissions
- **`session_id`**: `UUID` - Session identifier
- **`timezone`** / **`user_timezone`**: Timezone info
- **`conversation_id`**: MongoDB conversation ID
- **`history_items`**: Original conversation items for router lookup

### Other State

- **`usage_tracker`**: For recording LLM usage
- **`handoff_context`**: If handoff occurred (e.g., WorkoutCoach → WorkoutAnalyst)

---

## 2. What Agents Build Internally (Conversation Messages)

During execution, agents build a conversation message list that gets sent to the LLM:

```python
conversation = [
    SystemMessage(system_prompt),              # 1. Agent-specific system prompt
    SystemMessage(coach_notes_context),        # 2. Recent coach notes (last 3 days, max 5)
    SystemMessage(memory_context),             # 3. Long-term memories (semantic search, top 20)
    SystemMessage(previous_agents_context),    # 4. Previous agents' outputs (same turn)
    ...history_messages,                       # 5. Conversation history (filtered)
    HumanMessage(current_message)              # 6. Current user message
]
```

### Details:

1. **System Prompt**: Agent-specific instructions (from `get_system_prompt()`)

2. **Coach Notes**: Cross-agent context from last 3 days
   - Fetched from `_coach_note_repository`
   - Max 5 notes, all domains (cross-agent visibility)
   - Formatted as: `[date] (agent_name): content`

3. **Long-Term Memories**: Semantic search results
   - Top 20 memories, threshold 0.79
   - Searched via `memory_facade`
   - Respects `time_ranges` from router decision
   - Formatted as: `# Relevant Long-Term Memories`

4. **Previous Agents' Context**: Formatted from `agent_outputs`
   - Text summary (truncated to 500 chars)
   - Structured data (JSON from tool results)
   - Only includes outputs from OTHER agents (not self)
   - Format:
     ```
     # Context from Previous Agents
     ## AgentName:
     [summary text]
     **Structured Data:**
     ```json
     {tool_results}
     ```
     ```

5. **Conversation History**: Filtered messages from `state["messages"]`
   - **Removed**: ToolMessages, tool_calls, intermediate AIMessages
   - **Kept**: HumanMessages, final AIMessages
   - **Windowed**: Last N rounds (if `agent_context_window_size > 0`)
   - **Note**: Includes ALL messages from ALL previous agents/turns

6. **Current Message**: User's current input

---

## 3. What Agents Return (OUTPUT)

Agents return a `GraphState` update containing:

```python
GraphState(
    agent_outputs={
        agent_name: AgentOutput(
            agent_name="...",
            content="Final response text",
            tools_called=["tool1", "tool2"],
            execution_time_ms=1234,
            structured_data={
                "tool_results": {
                    "tool1": {...},  # Structured JSON from tool execution
                    "tool2": {...}
                }
            },
            tool_timestamps={
                "call_id_1": {"executedAt": ..., "returnedAt": ...}
            },
            new_messages=[AIMessage(...), ToolMessage(...), ...]
        )
    }
)
```

### Or for Handoff:

```python
Command(
    update={"handoff_context": HandoffContext(...)},
    goto="workout_analyst_execution"
)
```

---

## 4. What Accumulates Across Agents/Turns

### Persistent Across Turns (Session-Level)

- **`messages`**: Accumulates ALL HumanMessages + AIMessages from ALL turns
  - Uses `operator.add` - new messages appended
  - **Never cleared** during session
  - Includes messages from ALL agents

### Accumulates Within Turn (Execution-Level)

- **`agent_outputs`**: Dict accumulates as agents execute
  - Each agent adds its output: `state["agent_outputs"][agent_name] = output`
  - Later agents can read earlier agents' outputs via `state.get("agent_outputs")`

- **`routing_history`**: List of routing decisions

- **`total_execution_time_ms`**: Cumulative execution time

- **`errors`**: List of error messages

### Temporary (Not Passed to Next Agent)

- **`_memory_ids_used`**: Internal tracking for memory hit recording
- **`current_phase`**: Current execution phase
- **`pending_tool_calls`**: Tool management (internal)

---

## 5. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    GraphState (Shared)                      │
├─────────────────────────────────────────────────────────────┤
│ INPUT TO AGENT:                                              │
│  • messages: [ALL historical messages from ALL agents]      │
│  • current_message: Current user input                        │
│  • router_decision: Routing info                             │
│  • agent_outputs: {previous agents' outputs}                  │
│  • auth_ctx, session_id, timezone, etc.                      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│              Agent Builds Conversation                       │
│  [SystemPrompt] + [CoachNotes] + [Memories] +               │
│  [PreviousAgents] + [History] + [CurrentMessage]            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│              Agent Returns AgentOutput                       │
│  {                                                           │
│    agent_name, content, tools_called,                       │
│    structured_data, tool_timestamps, new_messages            │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│         State Accumulation (operator.add)                    │
│  • messages += [new AIMessage]                              │
│  • agent_outputs[agent_name] = output                        │
│  • routing_history += [decision]                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Key Insights

### ✅ What Works Well

- **All History Shared**: `messages` contains all conversation from all agents across all turns
- **Structured Data Passing**: Tool results passed as JSON in `AgentOutput.structured_data` for downstream agents
- **Cross-Agent Context**: Coach notes and memories provide cross-agent context
- **Previous Agent Outputs**: Later agents can see earlier agents' structured data in same turn

### ⚠️ Potential Issues

- **Irrelevant History**: When switching from Agent A (nutrition) to Agent B (workout), Agent B receives ALL history from Agent A, even if irrelevant
- **Token Waste**: Irrelevant history consumes context window
- **Cost**: More tokens = higher LLM costs
- **Confusion Risk**: Agents may misinterpret context from other domains

### 🔧 Filtering Limitations

- **Technical Filtering Only**: Removes tool noise (ToolMessages, tool_calls), NOT domain-specific content
- **Window Size Limiting**: Can limit to last N rounds, but still includes messages from ALL agents
- **No Domain Filtering**: No semantic filtering by agent domain/relevance

---

## 7. Example Scenario

### Turn 1: Nutrition Agent
```
User: "I ate pizza for lunch"
→ NutritionAgent executes
→ Returns: AgentOutput(content="Logged your pizza...", structured_data={meal_data})
→ messages += [HumanMessage("I ate pizza"), AIMessage("Logged your pizza...")]
```

### Turn 2: Workout Coach Agent
```
User: "I just ran 5 miles"
→ WorkoutCoachAgent executes
→ Receives:
  • messages: [HumanMessage("I ate pizza"), AIMessage("Logged your pizza..."), HumanMessage("I just ran 5 miles")]
  • agent_outputs: {} (no previous agents in this turn)
→ Builds conversation with:
  • System prompt (workout-specific)
  • Coach notes (if any)
  • Memories (workout-related)
  • History: [ALL messages including nutrition discussion]
  • Current message: "I just ran 5 miles"
→ Returns: AgentOutput(content="Great run!...", structured_data={workout_data})
→ messages += [AIMessage("Great run!...")]
```

**Note**: WorkoutCoachAgent receives nutrition history even though it's irrelevant to workout coaching.

---

## 8. Configuration Options

### History Filtering (`agent_history_filter_*`)

- **`agent_history_filter_enabled`**: Enable/disable filtering (default: True)
- **`agent_history_filter_tool_messages`**: Remove ToolMessages (default: True)
- **`agent_history_filter_tool_calls`**: Strip tool_calls from AIMessages (default: True)
- **`agent_history_filter_intermediate_ai`**: Remove intermediate AIMessages (default: True)

### Context Window (`agent_context_window_size`)

- **Default**: 0 (unlimited)
- **If > 0**: Limits to last N conversation rounds
- **Note**: Still includes messages from ALL agents, just limits how far back

---

## 9. Related Files

- **State Definition**: `src/domain/agents/langgraph/state.py`
- **Base Agent**: `src/domain/agents/langgraph/agents/base.py`
- **History Utils**: `src/domain/agents/langgraph/history_utils.py`
- **Graph Execution**: `src/domain/agents/langgraph/graph.py`
- **Aggregator**: `src/domain/agents/langgraph/nodes/aggregator.py`

---

## 10. Future Improvements

Potential enhancements to address irrelevant history:

1. **Agent-Specific History Filtering**: Filter messages by agent name before passing to each agent
2. **Domain-Aware Filtering**: Use semantic similarity to include only relevant messages
3. **Summary-Based Context**: Summarize previous agent outputs instead of raw messages
4. **Explicit Context Passing**: Use `agent_outputs` (structured data) instead of full message history

---

*Last Updated: Based on codebase analysis of `base.py` and `state.py`*
