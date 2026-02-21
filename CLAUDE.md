# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP (Model Context Protocol) server running in a Kali Linux Docker container. Provides AI assistants with access to security tools (nmap, nikto, sqlmap, gobuster, etc.) via SSE or stdio transport. Built on the `mcp` Python SDK using the low-level `Server` API.

## Commands

```bash
# Build and run Docker container (SSE mode, port 8000)
./run_docker.sh
# Or manually:
docker build -t kali-mcp-server . && docker run -p 8000:8000 kali-mcp-server

# Run all checks (type checking, linting, tests)
./run_tests.sh

# Individual dev commands
pyright                          # Type checking
ruff check .                     # Linting
ruff format .                    # Formatting
pytest                           # All tests
pytest tests/test_tools.py       # Single test file
pytest tests/test_tools.py::test_is_command_allowed  # Single test
pytest -k "session"              # Tests matching pattern

# Install dev dependencies
pip install -e ".[dev]"
```

## Architecture

### Core Files

- **`kali_mcp_server/server.py`** — Server setup, transport config (SSE via Starlette/uvicorn, or stdio), and **tool dispatch**. The `handle_tool_request()` function routes tool calls via an if/elif chain; `list_available_tools()` registers tool schemas with MCP.
- **`kali_mcp_server/tools.py`** — All tool implementations. Contains the `ALLOWED_COMMANDS` allowlist and session management backend.
- **`main.py`** and **`kali_mcp_server/__main__.py`** — Entry points (both delegate to `server.main()`).

### Adding a New Tool

New tools require changes in **three places**:
1. Implement the async function in `tools.py`
2. Add routing in `handle_tool_request()` in `server.py` (the if/elif dispatch)
3. Add the tool schema in `list_available_tools()` in `server.py` (returns `types.Tool` with `inputSchema`)
4. Import the function in `server.py`

### Command Security Model

`tools.py` defines `ALLOWED_COMMANDS` — a list of `(command_prefix, is_long_running)` tuples. The `is_command_allowed()` function checks commands against this allowlist by prefix matching. Shell metacharacters (`;`, `&`, `|`) are stripped from input. Long-running commands (nmap, nikto, etc.) are automatically backgrounded with output redirected to files.

### Session Management

File-based session system under `sessions/` directory. Each session gets a subdirectory with a `metadata.json` file storing name, description, target, timestamps, and command history. Active session tracked via `sessions/active_session.txt`.

### Transport Modes

- **SSE** (default): Starlette app with `/sse` endpoint and `/messages/` mount. Used with Claude Desktop.
- **stdio**: Direct stdin/stdout communication via `anyio.run`.

Select with `--transport sse|stdio` and `--port` (default 8000).

### Testing

Tests use `pytest-asyncio`. Server tests mock tool functions via `unittest.mock.patch` on `kali_mcp_server.server.<function_name>`. Tool tests call implementations directly (some require filesystem side effects like session creation).

### Docker

Based on `kalilinux/kali-rolling`. Runs as non-root `mcpuser`. Includes Go toolchain for `waybackurls`. Python runs in a venv at `/app/venv`. Pre-installs: nmap, metasploit, hydra, gobuster, dirb, nikto, sqlmap, testssl.sh, amass, httpx-toolkit, subfinder, gospider.

## Claude Desktop Integration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "kali-mcp-server": {
      "transport": "sse",
      "url": "http://localhost:8000/sse",
      "command": "docker run -p 8000:8000 kali-mcp-server"
    }
  }
}
```
