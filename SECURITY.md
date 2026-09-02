# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 0.4.x   | Yes       |
| < 0.4   | No        |

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Report security issues via GitHub's private vulnerability reporting, or email the maintainer directly. Please include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We will acknowledge receipt within 48 hours and aim to release a fix within 7 days for critical issues.

## Security Model

graphify is a **local development tool**. It runs as an AI coding assistant skill (Claude Code, Codex, OpenCode, Cursor, Gemini CLI, GitHub Copilot CLI, Aider, OpenClaw, Factory Droid, Trae, Hermes, Google Antigravity) and optionally as a local MCP stdio server. It makes no network calls during graph analysis - only during `ingest` (explicit URL fetch by the user).

### Threat Surface

| Vector | Mitigation |
|--------|-----------|
| SSRF via URL fetch | `security.validate_url()` allows only `http` and `https` schemes, blocks private/loopback/link-local IPs, and blocks cloud metadata endpoints. Redirect targets are re-validated. All fetch paths including tweet oEmbed go through `safe_fetch()`. |
| Oversized downloads | `safe_fetch()` streams responses and aborts at 50 MB. `safe_fetch_text()` aborts at 10 MB. |
| Non-2xx HTTP responses | `safe_fetch()` raises `HTTPError` on non-2xx status codes - error pages are not silently treated as content. |
| Path traversal in graph loading | Graph-loading call sites in `__main__.py`/`serve.py` validate in-line, not uniformly: `query` (`__main__.py`) and the MCP server's `_load_graph()` (`serve.py`) require the resolved path to exist AND end in `.json`; `path` and `explain` (`__main__.py`) only require the path to exist (no `.json` suffix check). `security.validate_graph_path()` implements a stricter check (containment inside `graphify-out/`) but is not currently called by any of these — it exists as a unit-tested helper only (`tests/test_security.py`). <!-- DOC-SYNC: 2026-08-15 재검증 — 2026-08-12 주석은 4개 호출부가 동일하게 exists+suffix를 검사한다고 오기재; 실측으로 `path`/`explain`은 exists()만 검사함을 확인, 정정 --> |
| XSS in graph HTML output | `security.sanitize_label()` strips control characters and caps at 256 chars; `export.py` additionally passes every node label, edge title, and community name through `html.escape()` before the vis.js renderer embeds them. |
| Prompt injection via node labels | `sanitize_label()` also applied to MCP text output - node labels from user-controlled source files cannot break the text format returned to agents. |
| YAML frontmatter injection | `_yaml_str()` escapes backslashes, double quotes, and newlines before embedding user-controlled strings (webpage titles, query questions) in YAML frontmatter. |
| Encoding crashes on source files | All tree-sitter byte slices decoded with `errors="replace"` - non-UTF-8 source files degrade gracefully instead of crashing extraction. |
| Symlink traversal | `detect()` takes a `follow_symlinks` parameter (default `False`, no CLI flag currently exposes `True`) that is passed straight through to `os.walk(..., followlinks=follow_symlinks)` in `detect.py`. |
| Corrupted graph.json | `_load_graph()` in `serve.py` wraps `json.JSONDecodeError` and prints a clear recovery message instead of crashing. |

### What graphify does NOT do

- Does not run a network listener (MCP server communicates over stdio only)
- Does not execute code from source files (tree-sitter parses ASTs - no eval/exec)
- Does not use `shell=True` in any subprocess call
- Does not store credentials or API keys

### Optional network calls

- `ingest` subcommand: fetches URLs explicitly provided by the user
- PDF extraction: reads local files only (pypdf does not make network calls)
- watch mode: local filesystem events only (watchdog does not make network calls)
