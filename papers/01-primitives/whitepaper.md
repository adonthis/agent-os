# Agent Primitives: A Type System for Safe Agent Failures

**Technical Report**

Imran Siddique  
Microsoft  
imran.siddique@microsoft.com

## Abstract

Current AI agent frameworks lack a principled type system for representing failure modes. Exceptions are either ignored, logged as strings, or wrapped in opaque error objects that lose semantic meaning across agent boundaries. We introduce **Agent Primitives**, a minimal type system that provides:

1. **Typed Failure Modes**: `AgentFailure` hierarchy distinguishing recoverable/unrecoverable errors
2. **State Preservation**: Failures carry agent state for debugging and replay
3. **Cross-Agent Propagation**: Failures maintain type information across agent boundaries

This foundational layer enables higher-level components (CMVK, Control Plane, SCAK) to reason about failures deterministically rather than heuristically.

## 1. Motivation

Consider a multi-agent system where Agent A calls Agent B:

```
Agent A → Agent B → Tool Execution → Failure
```

Without typed failures:
- Agent B catches `Exception`, logs "tool failed"
- Agent A receives `None` or a string error
- Root cause is lost; retry logic is blind

With Agent Primitives:
- `ToolExecutionFailure(tool="database", error="timeout", recoverable=True)`
- Agent A can inspect failure type and decide on retry strategy
- Full stack preserved for debugging

## 2. The Failure Hierarchy

```python
from agent_primitives import (
    AgentFailure,       # Base class
    ToolFailure,        # External tool errors
    PolicyFailure,      # Policy violations
    ContextFailure,     # Context/memory issues
    CommunicationFailure  # Inter-agent errors
)

class AgentFailure:
    message: str
    recoverable: bool
    agent_id: str
    timestamp: datetime
    context: dict  # Preserved state
    
    def to_signal(self) -> AgentSignal:
        """Convert to POSIX-style signal for kernel handling"""
```

## 3. Integration with Agent OS

Agent Primitives is Layer 1 of the Agent OS architecture:

```
L4: SCAK (Self-Correction) ───────────┐
L3: Control Plane (Governance) ───────┤
L2: IATP, AMB, ATR (Infrastructure) ──┤
L1: Primitives, CMVK, CaaS, EMK ──────┘ ← This paper
```

Higher layers depend on primitives for:
- **Control Plane**: Policy violations → `PolicyFailure`
- **SCAK**: Laziness detection → `RecoverableFailure`
- **IATP**: Trust boundary violations → `CommunicationFailure`

## 4. Design Principles

### 4.1 Scale by Subtraction

We avoid complex error hierarchies. The type system has exactly 4 failure categories, chosen for semantic clarity:

| Category | Cause | Action |
|----------|-------|--------|
| `ToolFailure` | External system | Retry with backoff |
| `PolicyFailure` | Governance violation | Log and block |
| `ContextFailure` | Memory/context | Purge and retry |
| `CommunicationFailure` | Inter-agent | Fallback agent |

### 4.2 Deterministic Mapping

Every failure maps to exactly one kernel signal:

```python
FAILURE_TO_SIGNAL = {
    ToolFailure: AgentSignal.SIGTOOL,
    PolicyFailure: AgentSignal.SIGPOLICY,  # → SIGKILL
    ContextFailure: AgentSignal.SIGCONTEXT,
    CommunicationFailure: AgentSignal.SIGCOMM
}
```

## 5. Usage

```python
from agent_primitives import ToolFailure, PolicyFailure

try:
    result = tool.execute(query)
except TimeoutError as e:
    raise ToolFailure(
        message=f"Database timeout: {e}",
        recoverable=True,
        tool_name="postgresql",
        context={"query": query}
    )

# In Control Plane:
if failure.recoverable:
    dispatcher.signal(agent_id, AgentSignal.SIGTOOL)  # Retry
else:
    dispatcher.signal(agent_id, AgentSignal.SIGKILL)  # Terminate
```

## 6. Related Work

Our approach differs from existing error handling:

- **LangChain**: Uses Python exceptions with string messages
- **AutoGen**: Wraps errors in conversation context
- **CrewAI**: Delegates to individual agents

Agent Primitives provides a **structured vocabulary** that enables kernel-level reasoning about failures.

## 7. Conclusion

Agent Primitives provides the minimal foundation for typed failures in agentic systems. By constraining the failure space to 4 categories with deterministic signal mappings, we enable higher-level components to reason about errors structurally rather than heuristically.

## References

- Siddique, I. (2026). Agent Control Plane: A Deterministic Kernel for Zero-Violation Governance. arXiv preprint.
- Siddique, I. (2026). SCAK: Automated Alignment via Differential Auditing. arXiv preprint.
- POSIX.1-2017. IEEE Std 1003.1-2017 (Signal Handling).

## Availability

Code: https://github.com/imran-siddique/agent-os  
PyPI: `pip install agent-primitives`

## License

MIT License
