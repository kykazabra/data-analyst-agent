# Data Analyst Agent

An LLM agent for exploratory data analysis that **writes its own reproducible Jupyter
notebook** while it works.

You give it a CSV or XLSX file and ask questions in plain language. The agent runs pandas
code to answer them — and every piece of code it executes, together with the output, is
appended to a notebook. When the conversation is over you are left with two things: the
answers, and a notebook you can re-run, verify and hand to someone else.

```python
from main import DataAnalyst

analyst = DataAnalyst(df_path="sales.csv", model="gpt-4o-mini")
analyst.talk("Which product categories lost revenue quarter over quarter?")
analyst.talk("Show the five worst performing regions for those categories")
# -> agent_results.ipynb now contains every step that produced those answers
```

## How it works

```
question ──► pandas dataframe agent ──► python_repl_ast tool ──► answer
                     │                          │
              CombinedMemory            CodeCallbackHandler
         (window + summary)                     │
                                         NotebookOperator ──► agent_results.ipynb
```

- **`NotebookOperator`** builds a notebook incrementally with `nbformat` and flushes it to
  disk after every cell, so the artifact stays valid even if the session dies halfway.
- **`CodeCallbackHandler`** hooks into the agent's tool lifecycle. On `on_tool_start` it
  captures the code the agent is about to run, on `on_tool_end` it pairs that code with the
  actual output and writes both into a cell. Only `python_repl_ast` calls are recorded —
  other tools are ignored, so the notebook stays a clean analysis rather than a trace log.
- **Memory is combined**: a sliding window of the last N exchanges plus an LLM-maintained
  running summary of the whole conversation. Both are injected into the agent prefix.

## Design decisions

**Why capture code through callbacks instead of asking the model for a notebook.**
Asking an LLM to produce a notebook at the end gives you code that was never executed —
it looks plausible and frequently does not run. Capturing the tool calls records only code
that actually ran, together with its real output. The notebook is evidence, not a retelling.

**Why write the file after every cell.** Analysis sessions are long and fail in the middle:
a rate limit, a bad query, a lost connection. Incremental flushing means a crashed session
still leaves a usable partial notebook.

**Why a combined memory rather than a plain buffer.** Data questions reference earlier
results a lot ("now the same for last year"). A window alone forgets the beginning of the
session; a summary alone loses the exact wording of recent turns. Keeping both costs one
extra LLM call per turn and removes most of the "what are we talking about" failures.

**Why `max_iterations=5`.** A pandas agent that cannot answer in a handful of steps is
usually stuck in a loop of failing code rather than converging. Capping it keeps a bad
question from burning the budget.

## Quickstart

```bash
pip install -r requirements.txt
cp credentials.example.json credentials.json   # put your API key there
python -c "from main import DataAnalyst; DataAnalyst('data.csv').talk('describe this dataset')"
```

`DataAnalyst` parameters: `df_path`, `model`, `cred_path`, `log_path`, `notebook_path`,
`thoughts_to_notebook`, `verbose`.

See [`examples/agent_results.ipynb`](examples/agent_results.ipynb) for a notebook produced
by an actual session.

## Limitations

- CSV (`;` delimited) and XLSX only.
- The agent executes generated Python in-process — run it on data and machines where that
  is acceptable.
- Prompt is pinned to Russian-language answers.
