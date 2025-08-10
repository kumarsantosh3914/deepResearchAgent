# Deep Research

A minimal, agent-driven research assistant with a Gradio UI. Enter a query and the system plans web searches, executes them in parallel, writes a long-form Markdown report, and emails the report via SendGrid.

The orchestration is in `research_manager.py` and the UI is in `deep_research.py`.

## Features

- Planner–Searcher–Writer pipeline driven by simple agent definitions
- Parallelized web search execution for faster turnarounds
- Long-form report generation in Markdown
- Optional email delivery (HTML) via SendGrid
- OpenAI Traces link printed for each run for easier debugging/observability

## How it works

High-level flow implemented by `ResearchManager.run()` in `research_manager.py`:

1. Trace started and a trace URL printed: `https://platform.openai.com/traces/trace?trace_id=...`.
2. Plan searches using `planner_agent` (`planner_agent.py`) which returns a `WebSearchPlan` (list of `WebSearchItem`).
3. Execute searches in parallel with `search_agent` (`search_agent.py`) using `WebSearchTool` and summarize results.
4. Synthesize a long-form Markdown report using `writer_agent` (`writer_agent.py`) which outputs a `ReportData` object.
5. Send the report as an HTML email via `email_agent` (`email_agent.py`) using SendGrid.
6. Stream status updates and finally the Markdown report back to the Gradio UI.

The UI in `deep_research.py` wires this `run(query)` coroutine to a Gradio `Blocks` app and streams the generated content to a `Markdown` component.

## Project structure

- `deep_research.py` — Gradio UI entry point, exposes `run(query)` and launches the app.
- `research_manager.py` — Orchestrates plan → search → write → email, streams progress.
- `planner_agent.py` — Defines `WebSearchItem`, `WebSearchPlan`, and the `planner_agent`.
- `search_agent.py` — Defines `search_agent` with `WebSearchTool` for summarizing results.
- `writer_agent.py` — Defines `ReportData` and the `writer_agent` that writes the report.
- `email_agent.py` — Defines a `send_email` tool and `email_agent` to deliver the report.
- `pyproject.toml` — Python 3.12+ project config and dependencies.
- `.env` — Environment variables (not committed). Loaded by `deep_research.py` via `python-dotenv`.

## Requirements

- Python 3.12+
- Windows, macOS, or Linux (tested on Windows)
- API keys:
  - OpenAI API key: required for agents and tracing
  - SendGrid API key: required if you enable email sending

## Quick start

You can install dependencies with either `uv` (recommended) or `pip`.

### Option A: Using uv (recommended)

1. Install uv (once):
   - `pip install uv` or follow uv docs for your system
2. Sync dependencies from `pyproject.toml`/`uv.lock`:
   - `uv sync`
3. Run the app:
   - `uv run python deep_research.py`

### Option B: Using pip

1. Create and activate a virtual environment (example on Windows PowerShell):
   - `python -m venv .venv`
   - `./.venv/Scripts/Activate.ps1`
2. Install dependencies:
   - `pip install -r <(python -c "import tomllib,sys;print('\n'.join(tomllib.load(open('pyproject.toml','rb'))['project']['dependencies']))")`
   - If the above one-liner is inconvenient, open `pyproject.toml` and install the listed packages manually with `pip install ...`.
3. Run the app:
   - `python deep_research.py`

When the app starts, your browser should open automatically to the Gradio interface.

## Environment variables

Create a `.env` file in the project root (`.env` is already in `.gitignore`). `deep_research.py` loads it automatically:

```
# Required for agent models and tracing
OPENAI_API_KEY=sk-...

# Required to send email via SendGrid
SENDGRID_API_KEY=SG....

# Optional: customize sender/recipient in email_agent.py if needed
# By default, the sender/recipient are set inline in email_agent.py
```

Note: Make sure the sender email is a verified sender in your SendGrid account.

## Running

- Start the app: `python deep_research.py`
- Enter a research query and press Run.
- Watch the streamed progress in the UI:
  - Trace URL printed first
  - Plan created
  - Searches executed in parallel
  - Report written
  - Email sent (if SENDGRID_API_KEY is present and valid)
- The final Markdown report appears in the UI and is also the final streamed chunk from `ResearchManager.run()`.

## Configuration knobs

You can tune the system without changing the overall flow:

- `planner_agent.py`
  - `HOW_MANY_SEARCHES` (default 5)
  - `model` for planning (currently `gpt-4o`)
- `search_agent.py`
  - `WebSearchTool(search_context_size="low")` — adjust context size if desired
  - `model_settings=ModelSettings(tool_choice="required")`
  - `model` for searching (currently `gpt-4o`)
- `writer_agent.py`
  - `INSTRUCTIONS` target length and style
  - `model` for writing (currently `gpt-4o`)
- `email_agent.py`
  - Update default sender/recipient addresses

## Observability & tracing

Each run prints a trace link like:

```
View trace: https://platform.openai.com/traces/trace?trace_id=<id>
```

Open it to inspect spans and tool calls for debugging.

## Troubleshooting

- No output / errors about missing keys
  - Ensure `.env` exists and `OPENAI_API_KEY` is set.
- Email not sent
  - Ensure `SENDGRID_API_KEY` is set, sender is verified, and recipient is allowed per your SendGrid plan.
- UI doesn’t open
  - Visit the printed Gradio URL manually or pass `server_port` to `ui.launch()` if a port conflict exists.
- Web search failures
  - Transient errors may be retried by re-running. Ensure network access.
