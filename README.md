# GitAssist — HITL Tool-Calling Agent

A GitHub intelligence assistant built on the **Model Context Protocol (MCP)**. Instead of guessing about repos from stale training data, the agent calls real tools against the live GitHub REST API — and a human approves every tool call before it fires.

## How it works

```
User
  ↓
LLM (Reason)
  ↓
Select Tool
  ↓
Human Approval
  ↓
MCP Tool
  ↓
GitHub API
  ↓
LLM Response
```

1. The user asks a question (e.g. *"what are the open issues on facebook/react?"*).
2. The LLM decides whether it needs a tool, and which one.
3. **Before any tool runs, the terminal prompts for human approval** (`y`/`n`) — a simple human-in-the-loop (HITL) safety gate.
4. If approved, the tool call goes out over MCP to `mcp_server.py`, which hits the real `api.github.com`.
5. The result is fed back to the LLM, which produces the final answer.
6. Conversation history is retained across turns, so you can say "now show me its contributors" without repeating the repo name.

## Architecture

| File | Role |
|------|------|

| `mcp_server.py` | An MCP server exposing 5 tools (`search_repositories`, `get_repo_details`, `list_open_issues`, `list_contributors`, `get_latest_release`), 1 resource (`github://repo/{owner}/{repo}/summary`), and 1 prompt (`issue_triage_prompt`) — all backed by real calls to the GitHub REST API. |

| `mcp_bridge.py` | Launches `mcp_server.py` as a subprocess, manages the MCP client session, and converts MCP tool schemas into OpenAI/Groq function-calling format. |

| `model_client.py` | Backend-agnostic LLM client — talks to a local **Ollama** model (`qwen2.5:3b` by default) via its OpenAI-compatible endpoint, or to **Groq** if `GROQ_API_KEY` is set. |

| `agent_core.py` | The main ReAct-style agent loop: reason → select tool → human approval → execute → respond. Also the CLI entry point. |

| `mcp_client_test.py` | A minimal standalone script that demonstrates the raw MCP handshake — `initialize()` → `list_tools()` → `call_tool()` — without any LLM involved. Good for sanity-checking the server on its own. |

## Setup

```bash
pip install -r requirements.txt
```

**Backend — pick one:**

- **Local (default):** install [Ollama](https://ollama.com), then `ollama pull qwen2.5:3b` and `ollama serve`.
- **Groq (cloud):** `export GROQ_API_KEY=your_key_here` (and optionally `export LLM_BACKEND=groq`).

**Optional:** `export GITHUB_TOKEN=your_token` to raise the GitHub API rate limit from 60 to 5,000 requests/hour.

## Usage

```bash
python agent_core.py
```

```
You : what are the open issues on facebook/react?

============================================================
 HUMAN-IN-THE-LOOP APPROVAL
============================================================
Tool Selected : list_open_issues
Arguments     : {'owner': 'facebook', 'repo': 'react'}
============================================================
Approve tool execution? (y/n): y
```

Type `quit` to exit.

To sanity-check the MCP server in isolation, without the agent loop:

```bash
python mcp_client_test.py
```

## Notes

- `env=os.environ.copy()` is passed explicitly when launching the MCP server subprocess — MCP's stdio transport does **not** inherit the parent process's environment by default, so without this the server would never see `GITHUB_TOKEN`.
- All GitHub calls are real, live HTTP requests — no mocked data.
