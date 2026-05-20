# Multi-step reasoning workflow

A multi-step reasoning agent solves problems that require planning across many interdependent steps, where the approach isn't obvious upfront and may need to backtrack. Unlike a simple ReAct loop, a multi-step reasoner explicitly manages its own reasoning state — tracking what it tried, what failed, and what alternative paths remain.

## When you need multi-step reasoning

| Signal | Example |
| ------ | ------- |
| The task has no obvious single correct path | "Find a bug in this codebase and fix it" |
| Early decisions affect which later steps are valid | "Write a migration plan, then execute it" |
| The agent might need to backtrack and try a different approach | "Solve this math problem" |
| Quality of the final answer matters more than speed | "Write a comprehensive literature review" |

If your task is linear (step A → B → C with no branching), use [plan-and-execute](../patterns/plan-and-execute.md) instead — it's simpler and faster.

## Reasoning architectures

### 1. ReAct + reflection (most common)

The agent runs a ReAct loop and then critiques its own output. If the critique finds problems, the agent revises and loops again.

```
┌────────────────────────────────────────────────────┐
│                                                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│  │  ReAct   │───▶│  Output  │───▶│  Self-critic │ │
│  │  loop    │    │          │    │              │ │
│  └──────────┘    └──────────┘    └──────┬───────┘ │
│        ▲                                │         │
│        └────────── Revise ─────────────┘         │
└────────────────────────────────────────────────────┘
```

### 2. Tree of Thoughts (exploration-heavy tasks)

The agent explores multiple reasoning paths in parallel before committing. Each branch is evaluated; the best branch continues.

```
                    ┌──────────────┐
                    │   Problem    │
                    └──────┬───────┘
                           │
           ┌────────────────┼────────────────┐
           │                │                │
      ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
      │ Path A  │      │ Path B  │      │ Path C  │
      │ Score:7 │      │ Score:9 │      │ Score:4 │
      └─────────┘      └────┬────┘      └─────────┘
                            │  (selected)
                     ┌──────▼───────┐
                     │   Solution   │
                     └──────────────┘
```

### 3. LATS (Language Agent Tree Search)

Combines Monte Carlo Tree Search with LLM reasoning. The agent selects actions based on a value function, explores promising branches, and backpropagates results to update branch estimates.

```
                ┌─────────┐
                │  Root   │  (initial state)
                └────┬────┘
                     │  select (UCB score)
              ┌──────┴──────┐
              │             │
         ┌────▼────┐   ┌────▼────┐
         │ Branch1 │   │ Branch2 │
         │  v=0.7  │   │  v=0.3  │
         └────┬────┘   └─────────┘
              │  expand
         ┌────▼────┐
         │  Child  │
         └────┬────┘
              │  simulate + backpropagate
         Update v for Branch1, Root
```

## Tools

### Stateful graph orchestration

**[LangGraph](https://github.com/langchain-ai/langgraph)** — Orchestrates multi-step reasoning as stateful, cyclical graphs with explicit branching and loop control. Tags: `planning` `orchestration` `python` `typescript`

LangGraph is the go-to for multi-step reasoning because it models the reasoning process as a graph where nodes are steps and edges are transitions. Conditional edges let the agent branch based on intermediate results. Cycles let it loop until a quality threshold is met.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class ReasoningState(TypedDict):
    problem: str
    current_solution: str
    critique: str
    iteration: int
    max_iterations: int

def solve(state: ReasoningState) -> ReasoningState:
    solution = llm.invoke(f"Solve: {state['problem']}\nPrevious attempt: {state['current_solution']}")
    return {**state, "current_solution": solution.content}

def critique(state: ReasoningState) -> ReasoningState:
    feedback = llm.invoke(f"Critique this solution: {state['current_solution']}\nProblem: {state['problem']}")
    return {**state, "critique": feedback.content, "iteration": state["iteration"] + 1}

def should_continue(state: ReasoningState) -> str:
    if state["iteration"] >= state["max_iterations"]:
        return "end"
    if "looks correct" in state["critique"].lower():
        return "end"
    return "solve"  # loop back

graph = StateGraph(ReasoningState)
graph.add_node("solve", solve)
graph.add_node("critique", critique)
graph.add_edge("solve", "critique")
graph.add_conditional_edges("critique", should_continue, {"solve": "solve", "end": END})
graph.set_entry_point("solve")

app = graph.compile()
result = app.invoke({"problem": "...", "current_solution": "", "critique": "", "iteration": 0, "max_iterations": 5})
```

### State machine tracking

**[Burr](https://github.com/dagworks-inc/burr)** — Manages multi-step agent state with built-in persistence, retries, and observability. Tags: `planning` `python` `stategraph`

Burr is excellent when you need to resume reasoning workflows after failures — the state is persisted between steps, so a crash doesn't mean starting over.

```python
from burr.core import action, State, ApplicationBuilder

@action(reads=["problem", "history"], writes=["solution"])
def reason(state: State) -> State:
    history_str = "\n".join(state["history"])
    solution = llm.invoke(f"Problem: {state['problem']}\nHistory:\n{history_str}\nNext step:")
    return state.update(solution=solution.content)

@action(reads=["solution", "history"], writes=["history", "is_done"])
def evaluate(state: State) -> State:
    new_history = state["history"] + [state["solution"]]
    is_done = len(new_history) >= 5 or "FINAL ANSWER" in state["solution"]
    return state.update(history=new_history, is_done=is_done)

app = (
    ApplicationBuilder()
    .with_actions(reason, evaluate)
    .with_transitions(("reason", "evaluate"), ("evaluate", "reason", lambda s: not s["is_done"]))
    .with_state(problem="...", history=[], solution="", is_done=False)
    .with_entrypoint("reason")
    .build()
)
action_name, result, state = app.run(halt_after=["evaluate"], inputs={"is_done": True})
```

### Tree-based reasoning

**[LATS](https://github.com/lapisrocks/LanguageAgentTreeSearch)** — Applies Monte Carlo tree search to LLM reasoning for tasks requiring exploration over many possible paths. Tags: `planning` `python` `research`

**[Tree of Thoughts](https://github.com/princeton-nlp/tree-of-thought-llm)** — Generates and evaluates multiple reasoning paths in parallel before selecting the best continuation. Tags: `planning` `python` `research`

---

## Pseudocode (ReAct + reflection with LangGraph)

```python
def multi_step_reasoning_agent(problem: str, max_iterations: int = 5) -> str:
    state = {
        "problem": problem,
        "current_solution": "",
        "critique": "",
        "iteration": 0,
        "max_iterations": max_iterations
    }

    for i in range(max_iterations):
        # Step 1: Generate or improve a solution
        solution = llm.invoke(
            f"Problem: {state['problem']}\n"
            f"Previous critique: {state['critique']}\n"
            "Generate an improved solution:"
        )
        state["current_solution"] = solution

        # Step 2: Critique the solution
        critique = llm.invoke(
            f"Solution: {state['current_solution']}\n"
            "Evaluate this solution. Is it correct and complete?\n"
            "Reply 'ACCEPTED' if it's correct, or explain specific issues."
        )

        if "ACCEPTED" in critique:
            return state["current_solution"]

        state["critique"] = critique
        state["iteration"] += 1

    # Return best solution after max iterations
    return state["current_solution"]
```

---

## Failure modes

| Failure | Cause | Fix |
| ------- | ----- | --- |
| Infinite refinement loops | Critique never says "accepted" | Add a maximum iteration count and return best solution |
| Sycophantic self-critique | LLM agrees with itself too easily | Use a separate model or persona for the critique step |
| Context window overflow | Long reasoning chains fill the context | Compress previous steps to summaries; keep only the last N full steps |
| Expensive tree exploration | ToT/LATS explores too many branches | Limit branching factor (max 3 branches) and depth (max 5 levels) |
| Lost state on failure | In-memory state lost on crash | Use Burr or LangGraph checkpointing for persistence |

---

## Production considerations

- **Cost control**: ReAct+reflection doubles LLM calls vs. a single pass. Tree-based methods can multiply costs 10–50×. Always instrument cost per reasoning run and set hard limits.
- **Observability**: Log each reasoning step with a trace ID. When a user reports a wrong answer, you need to replay the exact reasoning chain that produced it.
- **Timeouts**: Set wall-clock timeouts on the full reasoning loop (not just per-step). A reasoning agent stuck in a loop should fail fast, not hang for 10 minutes.
- **Human-in-the-loop**: For high-stakes decisions, pause the reasoning loop after N steps and ask a human to review before continuing. LangGraph supports this natively via interrupt points.

## Related pages

- [ReAct pattern](../patterns/react-pattern.md) — The foundational loop used inside multi-step reasoners
- [Reflection loop pattern](../patterns/reflection-loop.md) — The self-critique mechanism explained in depth
- [Planning skill](../skills/planning.md) — Tools and libraries for the planning layer
- [Research agent workflow](research-agent.md) — A concrete example using multi-step reasoning for research tasks
