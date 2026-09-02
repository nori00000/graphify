# Architecture

graphify is an AI coding assistant skill (Claude Code, Codex, OpenCode, Cursor, Gemini CLI, GitHub Copilot CLI, Aider, OpenClaw, Factory Droid, Trae, Hermes, Google Antigravity) backed by a Python library. The skill orchestrates the library; the library can be used standalone.

## Pipeline

```
detect()  →  transcribe_all()  →  extract()  →  build_from_json()  →  cluster()  →  god_nodes()/surprising_connections()/suggest_questions()  →  generate()  →  to_json()/to_html()/...
```

Each stage is a single function in its own module. They communicate through plain Python dicts and NetworkX graphs - no shared state, no side effects outside `graphify-out/`.

## Module responsibilities

| Module | Function | Input → Output |
|--------|----------|----------------|
| `detect.py` | `detect(root, *, follow_symlinks=False)` / `classify_file(path)` / `extract_pdf_text(path)` / `docx_to_markdown(path)` / `xlsx_to_markdown(path)` / `convert_office_file(path, out_dir)` / `save_manifest` / `load_manifest` / `detect_incremental` | directory → `{files: {code, doc, ...}, ...}` detection dict; also hosts file-type classification and office/PDF conversion helpers used by `extract.py`; `save_manifest` / `load_manifest` / `detect_incremental` also defined here (re-exported by `manifest.py` for backwards compatibility) |
| `extract.py` | `extract(paths)` / `collect_files(target, ...)` | list of file paths → `{nodes, edges}` dict; `collect_files` resolves a target directory to a sorted list of extractable file paths |
| `build.py` | `build_from_json(extraction, *, directed=False)` / `build(extractions, *, directed=False)` | extraction dict / list of extraction dicts → `nx.Graph`; `directed=True` produces a `nx.DiGraph` instead of the default `nx.Graph` |
| `cluster.py` | `cluster(G)` / `score_all(G, communities)` / `cohesion_score(G, community_nodes)` | graph → `dict[int, list[str]]` community map; `score_all` returns per-community cohesion scores `dict[int, float]`; `cohesion_score` returns a single community's internal edge density |
| `analyze.py` | `god_nodes(G)` / `surprising_connections(G, ...)` / `suggest_questions(G, ...)` / `graph_diff(G_old, G_new)` | graph → analysis lists; `graph_diff` returns added/removed nodes and edges between two graph snapshots |
| `report.py` | `generate(G, communities, ...)` | graph + analysis → GRAPH_REPORT.md string |
| `export.py` | `to_json / to_html / to_obsidian / to_canvas / to_svg / to_graphml / to_cypher / push_to_neo4j / prune_dangling_edges / attach_hyperedges` | graph → graph.json, graph.html, Obsidian vault, Obsidian canvas, graph.svg, graph.graphml, cypher.txt; `push_to_neo4j` pushes directly to a running instance; `prune_dangling_edges` removes edges whose endpoints are missing from the node set; `attach_hyperedges` merges hyperedge data from extraction into the graph |
| `ingest.py` | `ingest(url, ...)` / `save_query_result(question, answer, memory_dir, ...)` | URL → file saved to corpus dir; `save_query_result` saves a Q&A result as markdown in `graphify-out/memory/` so it is extracted into the graph on the next `--update` run |
| `cache.py` | `check_semantic_cache / save_semantic_cache` / `load_cached / save_cached / cached_files / clear_cache / file_hash` | `check_semantic_cache` takes a list of file paths and returns `(cached_nodes, cached_edges, cached_hyperedges, uncached_files)`; `save_semantic_cache` groups nodes/edges/hyperedges by `source_file` and saves one cache entry per file; `load_cached` / `save_cached` read and write per-file SHA256 extraction cache; `cached_files` returns the set of SHA256 hash digests for all currently cached entries; `clear_cache` removes all cache entries; `file_hash` computes a SHA256 hex digest for a file |
| `security.py` | validation helpers | URL / path / label → validated or raises |
| `validate.py` | `validate_extraction(data)` / `assert_valid(data)` | `validate_extraction` returns `list[str]` of error strings (empty list = valid, does not raise) — called by `build_from_json()` before graph assembly; `assert_valid` raises `ValueError` with all errors if the extraction dict is invalid — convenience wrapper for external callers, not called by the build functions |
| `serve.py` | `serve(graph_path)` | graph file path → MCP stdio server |
| `transcribe.py` | `transcribe_all(paths, ...)` / `transcribe(path, ...)` / `download_audio(url, output_dir)` / `is_url(path)` / `build_whisper_prompt(god_nodes)` | video/audio paths → transcript dicts (faster-whisper); `download_audio` fetches audio from a URL via yt-dlp; `is_url` detects whether a path is a URL; `build_whisper_prompt` generates a domain-aware initial prompt from god node labels |
| `watch.py` | `watch(watch_path, ...)` | directory → auto-rebuilds graph on file changes |
| `benchmark.py` | `run_benchmark(graph_path, corpus_words=None, questions=None)` / `print_benchmark(result)` | graph file → corpus vs subgraph token comparison; `corpus_words` overrides corpus word count (default: computed from graph); `questions` is an optional list of sample queries to benchmark; `print_benchmark` prints the human-readable token reduction report |
| `wiki.py` | `to_wiki(G, communities, ...)` | graph + community map → Wikipedia-style markdown wiki (index.md + article per community / god node) |
| `hooks.py` | `install(path, ...)` / `uninstall(path)` / `status(path)` | git repo path → installs/removes post-commit and post-checkout hooks |
| `manifest.py` | re-exports `save_manifest`, `load_manifest`, `detect_incremental` | backwards-compatibility shim — delegates to `detect.py` |
| `__init__.py` | `__getattr__` lazy re-exports | package import → public API surface for standalone library use (`extract`, `build_from_json`, `cluster`, `god_nodes`, `generate`, `to_json`, `to_wiki`, etc.); imports are deferred so `graphify install` works before heavy deps are present |
| `__main__.py` | `main()` | CLI entry point — parses argv, routes all subcommands (`install`, `claude`, `gemini`, `cursor`, `hook`, `query`, `path`, `explain`, `add`, `watch`, `update`, `cluster-only`, `antigravity`, etc.), and holds `_PLATFORM_CONFIG` with per-platform skill destinations; the MCP server is started via `python -m graphify.serve` (not a `graphify serve` subcommand) |

## Extraction output schema

Every extractor returns:

```json
{
  "nodes": [
    {
      "id": "unique_string",
      "label": "human name",
      "file_type": "code|document|paper|image|rationale",
      "source_file": "path",
      "source_location": "L42"
    }
  ],
  "edges": [
    {"source": "id_a", "target": "id_b", "relation": "calls|imports|uses|...", "confidence": "EXTRACTED|INFERRED|AMBIGUOUS", "source_file": "path"}
  ]
}
```

Required node fields: `id`, `label`, `file_type`, `source_file`. `source_location` is optional.
Required edge fields: `source`, `target`, `relation`, `confidence`, `source_file`.

`to_json()` in `export.py` writes a `norm_label` field onto each node at serialisation time (NFKD-normalised, lowercased label) for diacritic-insensitive search. This field is not part of the extraction schema and is not validated by `assert_valid()`.

`validate_extraction(data)` in `validate.py` is called by `build_from_json()` before graph assembly; it returns a `list[str]` of error strings (empty = valid) and does not raise. `assert_valid(data)` is a convenience wrapper for external callers that raises `ValueError` if errors are found — it is **not** called by the build functions.

## Confidence labels

| Label | Meaning |
|-------|---------|
| `EXTRACTED` | Relationship is explicitly stated in the source (e.g., an import statement, a direct call) |
| `INFERRED` | Relationship is a reasonable deduction (e.g., call-graph second pass, co-occurrence in context) |
| `AMBIGUOUS` | Relationship is uncertain; flagged for human review in GRAPH_REPORT.md |

## Adding a new language extractor

1. Add a `extract_<lang>(path: Path) -> dict` function in `extract.py` following the existing pattern (tree-sitter parse → walk nodes → collect `nodes` and `edges` → call-graph second pass for INFERRED `calls` edges).
2. Register the file suffix in `extract()` dispatch table (`_DISPATCH` dict). **Exception:** compound extensions like `.blade.php` cannot be keyed by `Path.suffix` (which returns `.php`), so they are handled by a separate `path.name.endswith()` check before the dispatch lookup — no `_DISPATCH` entry is needed for those.
3. Add the suffix to `CODE_EXTENSIONS` in `detect.py`. `watch.py` derives `_WATCHED_EXTENSIONS` automatically from `detect.py`'s constants — no manual edit needed there.
4. Add the tree-sitter package to `pyproject.toml` dependencies. **Exception:** if the language is extracted via regex or reuses an existing extractor (e.g. `.vue` and `.svelte` reuse `extract_js`; `.dart` uses a regex extractor), no new tree-sitter package is needed.
5. Add a fixture file to `tests/fixtures/` and tests to `tests/test_languages.py`.

## Security

All external input passes through `graphify/security.py` before use:

- URLs → `validate_url()` (http/https only) + `_NoFileRedirectHandler` (blocks file:// redirects)
- Fetched content → `safe_fetch()` / `safe_fetch_text()` (size cap, timeout)
- Graph file paths → checks vary by call site, all in-line (none call `security.validate_graph_path()`): `query` (`__main__.py`) and the MCP server's `_load_graph()` (`serve.py`) check both that the path exists AND ends in `.json`; `path` and `explain` (`__main__.py`) check only that the path exists (no suffix check) — so graphs outside `graphify-out/` can still be loaded by any of the four. `security.validate_graph_path()` implements a stricter containment-inside-`graphify-out/` check but is not wired into any of these call sites — it is exercised only by `tests/test_security.py` <!-- DOC-SYNC: 2026-08-15 재검증 — 2026-08-12 주석은 `path`/`explain`이 `query`/`serve.py`와 동일하게 .json 접미사까지 검사한다고 오기재; 실측(__main__.py:938-1050)으로는 `path`/`explain`은 exists()만 검사, suffix 검사 없음. 정정. -->
- Node labels → `sanitize_label()` (strips control chars, caps 256 chars); HTML output additionally passes labels through `html.escape()` in `export.py`

See `SECURITY.md` for the full threat model.

## Testing

Tests live under `tests/`. Each module has a corresponding test file (e.g. `test_build.py`, `test_cluster.py`, `test_export.py`, `test_extract.py`, `test_analyze.py`, `test_benchmark.py`, `test_cache.py`, `test_detect.py`). Additional cross-cutting test files cover integration scenarios: `test_pipeline.py`, `test_multilang.py`, `test_languages.py`, `test_semantic_similarity.py`, `test_hypergraph.py`, `test_rationale.py`, `test_install.py`, `test_claude_md.py`, `test_confidence.py`, `test_hooks.py`, `test_ingest.py`, `test_report.py`, `test_security.py`, `test_serve.py`, `test_transcribe.py`, `test_validate.py`, `test_watch.py`, `test_wiki.py`. Run with:

```bash
pytest tests/ -q --tb=short
```

All tests are pure unit tests - no network calls, no file system side effects outside `tmp_path`.
