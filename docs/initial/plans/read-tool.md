# Plan: Object Reader (`read` tool)

## Context

The assistant currently has no way to fetch Opik entities (traces, spans, datasets, etc.). The `read` tool is the primary data-access tool: it fetches any Opik entity via the backend REST API, returns an adaptively compressed representation to the LLM, and caches the full JSON in the session for drill-down via future `jq`/`search` tools.

Design doc reference: `docs/initial/design-doc.md` — sections "Object Reader", "Tool Inventory", and "Appendix F: Entity Compression Algorithms".

## Approach

### Tool context injection — closure factory

The existing tool system passes only LLM kwargs to tool functions (`tool_def.fn(**tool_use.input)`). The `read` tool needs session (for caching) and user credentials (for API auth).

The `BaseAgent.tools()` docstring already anticipates this: *"Takes context because some tools are closures that capture per-request state."*

So we use a factory function `make_read_tool(ctx)` that returns a `ToolDef` whose inner function closes over the context. No changes needed to `@tool` decorator, `execute_tool`, or `ToolExecutor`.

We add a `ToolContext` model to `AgentContext` to thread session + auth data from the chat router to the agent's `tools()` method.

## Implementation Steps

### Step 1: Add `httpx` dependency

```
uv add httpx
```

### Step 2: Create type definitions — `src/ollie_assist/types/opik.py`

- `EntityType(StrEnum)` — `trace`, `span`, `dataset`, `dataset_item`, `project`, `thread`
- `CompressionTier(StrEnum)` — `FULL`, `MEDIUM`, `SKELETON`, `SUMMARY`

### Step 3: Add `ToolContext` to agent types — `src/ollie_assist/types/agent.py`

```python
class ToolContext(BaseModel):
    """Runtime context injected into tools that need backend/session access."""
    model_config = {"arbitrary_types_allowed": True}

    session: Any  # typed as Any to avoid circular import with agent_core.session
    session_token: str | None = None
    workspace_name: str | None = None
    opik_base_url: str = ""
```

Add `tool_context: ToolContext | None = None` field to `AgentContext`.

### Step 4: Create Opik API client — `src/ollie_assist/opik_client.py`

Class `OpikClient` wrapping `httpx.AsyncClient`. Constructor takes `base_url`, `session_token`, `workspace`. Builds auth headers/cookies once.

Methods (all return `dict[str, Any]`):

| Method | HTTP | Endpoint | Params |
|--------|------|----------|--------|
| `get_trace(trace_id)` | GET | `v1/private/traces/{id}` | query: `strip_attachments=false` |
| `get_spans_for_trace(trace_id, project_id)` | POST | `v1/private/spans/search` | body: `{trace_id, project_id, limit: 2000, truncate: false}` (NDJSON response) |
| `get_span(span_id)` | GET | `v1/private/spans/{id}` | query: `strip_attachments=false` |
| `get_project(project_id)` | GET | `v1/private/projects/{id}` | — |
| `get_project_stats(project_name)` | GET | `v1/private/projects/stats` | query: `name={name}&size=1` |
| `get_dataset(dataset_name)` | POST | `v1/private/datasets/retrieve` | body: `{"dataset_name": name}` |
| `get_dataset_items(dataset_name, limit=5)` | POST | `v1/private/datasets/items/stream` | body: `{"dataset_name": name, "steam_limit": limit}` (NDJSON response) |
| `get_dataset_item(item_id)` | GET | `v1/private/datasets/items/{item_id}` | — |
| `get_thread(thread_id, project_name?, project_id?)` | POST | `v1/private/traces/threads/retrieve` | body: `{"thread_id": id, "truncate": false}` + optional project fields |

**Spans**: Uses `POST /v1/private/spans/search` (NDJSON streaming) instead of `GET /v1/private/spans` because the GET endpoint requires `project_name` or `project_id` alongside `trace_id`. The `project_id` is extracted from the trace response.

**Dataset items stream**: Returns NDJSON (newline-delimited JSON). Note the API typo: param is `steam_limit` not `stream_limit`.

Custom `OpikApiError` exception for non-2xx responses.

**Error messages returned to the LLM:**

| Condition | Message |
|-----------|---------|
| 404 on trace/span/project/dataset_item/thread | `"Not found: {entity_type} with id '{id}'. Verify the ID is a valid UUID and belongs to the current workspace."` |
| 404 on dataset | `"Not found: dataset with name '{id}'. Verify the dataset name is correct (datasets are looked up by name, not ID)."` |
| Empty spans for trace | Not an error — return trace with empty spans list. |
| Empty dataset items stream | Not an error — include `"items": []`. |
| Any other HTTP error | `"Failed to fetch {entity_type} '{id}': {backend_error_messages}. (HTTP {status})"` |
| Network/timeout error | `"Could not reach the Opik backend to fetch {entity_type} '{id}'. The service may be temporarily unavailable."` |
| Missing OPIK_URL_OVERRIDE | `"Opik backend URL is not configured. Set the OPIK_URL_OVERRIDE environment variable."` |

### Step 5: Create compression module — `src/ollie_assist/tools/compression.py`

Pure functions, no I/O. Reuse patterns from Appendix F.

**Utilities:**

- `estimate_tokens(text: str) -> int` — `len(text) // 4`
- `size_header(entity_type, entity_id, tier, returned_tokens, full_tokens) -> str`
- `truncate_strings(obj, path="", threshold=200) -> Any` — recursive traversal, tracks jq path, appends `[TRUNCATED N chars — use jq('path') to see full]`

**Trace-specific:**

- `build_span_tree(trace, spans) -> str` — tree diagram with `├──`/`└──` connectors. Each node: `"name" [type] id={span_id} (duration ERROR?)`. Span IDs included at all tiers so the agent can drill into specific spans.
- `compress_trace(trace, spans, max_tokens?) -> tuple[str, CompressionTier]`:
  - FULL (<8K tokens): complete JSON
  - MEDIUM (8K–50K): span tree skeleton + truncated JSON
  - SKELETON (50K+): span tree only + count

**Dataset-specific:**

- `compress_dataset(dataset, items) -> tuple[str, CompressionTier]` — always SUMMARY: metadata + 5 sample items (labeled `sample_items`) with truncated strings + note about total items and how to drill down.

**Generic (span, project, dataset_item, thread):**

- `compress_generic(entity, entity_type, entity_id, max_tokens?) -> tuple[str, CompressionTier]`:
  - FULL (<8K tokens): complete JSON
  - MEDIUM (8K+): JSON with strings truncated at 200 chars

### Step 6: Create read tool — `src/ollie_assist/tools/read.py`

Factory function `make_read_tool(ctx: ToolContext) -> ToolDef`.

Inner async function `read(entity_type: str, id: str, max_tokens: int | None = None) -> str`:

1. Validate entity_type, check opik_base_url configured
2. Check session cache (`{entity_type}:{id}`)
3. If miss: create `OpikClient`, fetch entity:
   - **trace**: `get_trace(id)` + `get_spans_for_trace(id, project_id)` → cache as `{"trace": {...}, "spans": [...]}`
   - **dataset**: `get_dataset(id)` + `get_dataset_items(id, limit=5)` → cache as `{"dataset": {...}, "items": [...]}`
   - **project**: `get_project(id)` + `get_project_stats(name)` → merge stats into project dict
   - **thread**: `get_thread(id)` → cache the response
   - **span/dataset_item**: single GET → cache the response
4. Cache full JSON in `ctx.session.cache_object(key, raw_data)`
5. Dispatch to appropriate compression function
6. Prepend size header, return

Tool description stored as the `read()` function's docstring, extracted via `inspect.cleandoc()`.

Build `ToolDef` manually using `schema_from_annotations`.

**Important**: `read.py` must NOT use `from __future__ import annotations` — the closure's type hints need to be available at runtime for `schema_from_annotations`.

Errors logged at WARNING (API errors) or ERROR with traceback (network errors).

### Step 7: Wire up context — modify existing files

**`src/ollie_assist/routers/chat.py`**: Construct `ToolContext` from `session`, `user`, and `settings.opik_url_override`, pass it into `AgentContext`.

**`src/ollie_assist/agents/assistant/agent.py`**: In `tools()`, call `make_read_tool(context.tool_context)` and append to tool list.

**`src/ollie_assist/tools/decorator.py`**: Rename `_schema_from_annotations` → `schema_from_annotations` (public API).

**`src/ollie_assist/agent_core/client.py`**: Use `model_dump(exclude_none=True)` for assistant message blocks to avoid Anthropic API rejecting extra fields (citations, caller, parsed_output).

## Files Created

| File | Purpose |
|------|---------|
| `src/ollie_assist/types/opik.py` | `EntityType`, `CompressionTier` enums |
| `src/ollie_assist/opik_client.py` | HTTP client for Opik backend API |
| `src/ollie_assist/tools/compression.py` | All compression algorithms |
| `src/ollie_assist/tools/read.py` | `make_read_tool` factory |
| `scripts/test_read_tool.py` | Standalone test script for manual testing |

## Files Modified

| File | Change |
|------|--------|
| `src/ollie_assist/types/agent.py` | Add `ToolContext`, add field to `AgentContext` |
| `src/ollie_assist/tools/decorator.py` | Make `schema_from_annotations` public |
| `src/ollie_assist/agents/assistant/agent.py` | Register `read` tool |
| `src/ollie_assist/routers/chat.py` | Construct + pass `ToolContext` |
| `src/ollie_assist/agent_core/client.py` | `exclude_none=True` for assistant block serialization |
| `pyproject.toml` | Add `httpx` + `respx` dependencies |

## Key Reuse

- `Session.cache_object` / `get_cached_object` — already implemented in `session.py`
- `ToolDef` model — existing in `types/agent.py`
- `schema_from_annotations` — existing in `tools/decorator.py` (renamed from `_schema_from_annotations`)
- Closure-in-`tools()` pattern — anticipated by `BaseAgent.tools()` docstring

## Verification

1. **Unit tests** (`tests/test_compression.py`):
   - Token estimation, string truncation with jq paths, span tree rendering (with IDs)
   - Each compression tier for traces (FULL/MEDIUM/SKELETON)
   - Dataset SUMMARY compression with sample note
   - Generic FULL/MEDIUM compression
   - Size header format

2. **Unit tests** (`tests/test_opik_client.py`):
   - Auth header/cookie forwarding
   - NDJSON parsing for spans search and dataset items
   - Project stats fetching
   - Error handling (4xx/5xx → `OpikApiError`)

3. **Unit tests** (`tests/test_read_tool.py`):
   - Cache hit/miss behavior
   - Size header presence in output
   - Invalid entity type → error
   - Tool schema correctness (has `entity_type`, `id`, `max_tokens`)
   - Error message formatting (404, 500, network)

4. **Manual testing**: `scripts/test_read_tool.py` for testing against a running Opik backend.

## Decisions Made During Implementation

1. **Circular import avoidance**: `ToolContext.session` typed as `Any` instead of `Session` to break the `types/agent.py` ↔ `agent_core/session.py` cycle.
2. **Spans endpoint**: Changed from `GET /v1/private/spans` (requires project_name/project_id, returns paginated JSON) to `POST /v1/private/spans/search` (accepts trace_id + project_id, returns NDJSON). The `project_id` is extracted from the trace response.
3. **Project stats enrichment**: `GET /v1/private/projects/stats?name=...&size=1` fetched alongside the project and merged into the response. Includes trace_count, thread_count, error_count, duration percentiles, cost, and usage.
4. **Assistant block serialization**: `model_dump(exclude_none=True)` required to prevent Anthropic API errors from extra SDK fields (citations, caller, parsed_output).
5. **Dataset sample labeling**: Items labeled as `sample_items` with an explicit note about the sample size and how to drill into individual items.
6. **Span tree IDs**: Span IDs included in tree rendering at all compression tiers so the agent can always identify spans for drill-down.
