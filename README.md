# AI Research & Essay Writing Assistant

An autonomous, multi-step AI agent built with **LangGraph** that takes any topic, researches it on the web, drafts a 5-paragraph essay, critiques itself, and iteratively improves the draft over multiple revisions.

The agent follows a *Plan → Research → Write → Reflect → Research again → Rewrite* loop, mimicking how a human writer would approach a piece — outline first, gather sources, draft, get feedback, revise.

---

## Architecture

The agent is implemented as a state graph with five nodes that pass a shared `AgentState` between them:

```
                ┌─────────────┐
                │   planner   │   ← creates a high-level outline
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ research_plan│   ← generates 3 search queries, fetches results via Tavily
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
        ┌──────►│  generate   │   ← writes / rewrites the essay draft
        │       └──────┬──────┘
        │              │
        │     should_continue?
        │       │             │
        │       │ revisions   │ revisions
        │       │ remaining   │ exhausted
        │       ▼             ▼
        │  ┌─────────┐       END
        │  │ reflect │       ← critiques the current draft
        │  └────┬────┘
        │       │
        │       ▼
        │  ┌──────────────────┐
        └──┤ research_critique│  ← researches based on the critique
           └──────────────────┘
```

### Nodes

| Node | Role |
|---|---|
| `planner` | Generates a structured outline for the essay topic. |
| `research_plan` | Produces up to 3 search queries from the task and fetches snippets via the Tavily API. |
| `generate` | Writes (or rewrites) a 5-paragraph essay using the plan and accumulated research. |
| `reflect` | Acts as a critical teacher, producing detailed critique and recommendations. |
| `research_critique` | Generates new search queries based on the critique and fetches additional context. |

### Shared State

```python
class AgentState(TypedDict):
    task: str               # the user's topic
    plan: str               # the outline
    draft: str              # the latest essay draft
    critique: str           # feedback from the reflect node
    content: List[str]      # accumulated research snippets
    revision_number: int    # how many drafts have been written
    max_revisions: int      # stop condition
```

### Looping & Termination

After every `generate` step, the conditional edge `should_continue` checks whether `revision_number` has exceeded `max_revisions`. If yes, the graph ends. If no, the draft is sent back through `reflect → research_critique → generate` for another pass.

Persistence between steps is handled by an in-memory **SQLite checkpointer** (`SqliteSaver`), which lets you resume runs by `thread_id`.

---

## Tech Stack

- **LangGraph** — graph-based orchestration of the agent's state machine
- **LangChain** — message abstractions and structured output
- **Anthropic Claude** (`claude-haiku-4-5`) — the underlying LLM via `langchain-anthropic`
- **Tavily** — web search API for the research nodes
- **Pydantic** — schema for structured query generation
- **SQLite** — in-memory checkpointing of agent state

---

### Streaming intermediate steps

The file includes a commented-out streaming block. Uncomment it to watch each node's output as the graph runs — useful for debugging:

```python
for s in graph.stream({
    'task': "What is radiation sickness?",
    "max_revisions": 2,
    "revision_number": 1,
}, thread):
    pprint.pprint(s)
```
