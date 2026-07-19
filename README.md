# Agent for Blogs

A multi-agent blog-writing pipeline built with [Google's Agent Development Kit (ADK)](https://google.github.io/adk-docs/). Give it a topic, and it plans an outline, writes a full technical blog post, validates its own work, and retries automatically if the output doesn't meet quality checks.

## How it works

The pipeline is composed of nested agents:

```
Blogger (root agent)
├── RobustBlogPlanner (LoopAgent, max 3 iterations)
│   ├── blog_planner        → drafts a Markdown outline
│   └── OutlineValidationChecker → checks the outline; "ok" or "retry"
│
└── RobustBlogWriter (LoopAgent, max 3 iterations)
    ├── blog_writer          → writes the full article from the outline
    └── BlogPostValidationChecker → checks the post; "ok" or "retry"
```

The root agent (`Blogger`) exposes the planner and writer as tools it can call directly. Given a topic, it:

1. Calls the planner tool to generate an outline (stored in state as `blog_outline`).
2. Calls the writer tool to produce the full draft (stored in state as `blog_post`).
3. Wraps up with 3 alternate titles and 2 tweet-length hooks.

Each `LoopAgent` retries its inner agent up to `max_iterations` times if the validator responds `"retry"`, and stops early once it responds `"ok"`.

## Project structure

```
Agent_for_blogs/
├── __init__.py       # exposes agent.py so ADK can discover root_agent
├── agent.py          # all agent definitions live here
├── .env              # holds your API key (not committed to git)
└── requirements.txt
```

## Requirements

- Python 3.13 (managed via `pyenv`)
- A Google AI Studio / Gemini API key ([get one here](https://ai.google.dev/gemini-api/docs/api-key))

## Setup

1. **Set your Python version** (if not already active):
   ```bash
   pyenv local 3.13.0
   ```

2. **Create and activate your virtual environment**, then install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. **Add your API key.** Create a `.env` file inside `Agent_for_blogs/`:
   ```
   GOOGLE_API_KEY=your_actual_api_key_here
   ```

4. **(Optional) Choose a model.** By default the agents use `gemini-flash-latest`. Override it via the `.env` file:
   ```
   MODEL=gemini-1.5-pro
   ```

## Running

Start the ADK dev UI from inside the project directory:

```bash
adk web
```

Then open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser, select the `Agent_for_blogs` app, and enter a blog topic in the chat to kick off the pipeline.

## Notes

- If the trend/context tool fails or times out, the agent is instructed to continue with a sensible outline and draft anyway rather than failing outright.
- The `output_key` fields (`blog_outline`, `blog_post`, `validation_result`) control how sub-agents share data through shared session state — downstream agents read these keys directly from their instructions.
- `.env` should never be committed to version control; add it to `.gitignore`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `SyntaxError` on module load | Check for typos in agent kwargs/brackets in `agent.py` |
| `404` on `/dev/apps/.../build_graph` | `root_agent` not discoverable — check `__init__.py` imports `agent`, and that agent classes use `__init__` (not `_init_`) |
| `ValueError: No API key was provided` | Missing or misplaced `.env`, or wrong env var name (`GOOGLE_API_KEY`) |
