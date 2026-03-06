# Test Coverage Analysis

## Current State

The project has **1 test file** (`tests/test_queue.py`) containing **7 test cases**, all targeting the task queue module. Estimated code coverage is **~20%** — only `src/queue/task_queue.py` (327 lines) is tested out of ~1,635 total lines of source code.

### What's Tested

| Module | File | Lines | Tests | Coverage |
|--------|------|-------|-------|----------|
| Task Queue | `src/queue/task_queue.py` | 327 | 7 tests | Partial |

The existing tests cover: task submission, retrieval, priority ordering, completion, failure/retry, batch submission, and statistics.

### What's Not Tested

| Module | File | Lines | Tests | Coverage |
|--------|------|-------|-------|----------|
| REST API Client | `src/api/client.py` | 368 | 0 | None |
| OAuth Auth | `src/auth/oauth.py` | 319 | 0 | None |
| Salesforce Agent | `src/agents/salesforce_agent.py` | 330 | 0 | None |
| Bulk API Client | `src/api/bulk.py` | 279 | 0 | None |
| CLI Entry Point | `src/main.py` | 179 | 0 | None |
| Daemon | `src/daemon.py` | 218 | 0 | None |
| SFDX Runner | `src/run_sfdx.py` | 159 | 0 | None |

---

## Recommended Improvements (by priority)

### 1. OAuth Authentication — CRITICAL

**File:** `src/auth/oauth.py` (319 lines, 0 tests)

This is the foundation every other module depends on. A broken auth module means the entire agent is non-functional.

**Tests to add (`tests/test_auth.py`):**

- **`TokenInfo.is_expired`** — Unit test the 5-minute buffer expiration logic. This is a pure calculation with no external dependencies, making it the easiest and highest-value test to write.
- **`TokenInfo.to_dict` / `from_dict`** — Round-trip serialization. Important because token caching depends on this.
- **`SalesforceAuth._username_password_flow`** — Mock `requests.post` to verify the correct payload is sent (grant_type, client_id, password+security_token concatenation) and that `TokenInfo` is constructed correctly from the response.
- **`SalesforceAuth._refresh_token`** — Mock the refresh endpoint. Verify the existing refresh token is preserved while the access token is updated.
- **`SalesforceAuth.authenticate` fallback** — Verify that when refresh fails, it falls back to username-password flow.
- **`SalesforceAuth._cache_token` / `_load_cached_token`** — Test file creation, `0o600` permissions, and loading from a temp directory. Test graceful handling of missing/corrupt cache files.
- **`SalesforceAuth.login_url`** — Verify URL selection for sandbox (`test.salesforce.com`), production (`login.salesforce.com`), and custom domains (`*.my.salesforce.com`).
- **`SalesforceAuth.from_sfdx`** — Mock the `~/.sfdx/` directory with a fixture. Test file discovery, JSON parsing, and token construction.
- **`SalesforceAuth.revoke`** — Verify the revoke endpoint is called and the cached token file is deleted.
- **Error paths** — Authentication failure (non-200 response) should raise `AuthenticationError` with the error description.

### 2. REST API Client — CRITICAL

**File:** `src/api/client.py` (368 lines, 0 tests)

This is the module that actually talks to Salesforce. Every agent operation flows through it.

**Tests to add (`tests/test_client.py`):**

- **CRUD operations (`create`, `get`, `update`, `delete`, `upsert`)** — Use `unittest.mock.patch` or `responses` library to mock `requests.Session`. Verify correct HTTP method, URL construction, headers, and payload for each operation. Verify return values (record ID for create, dict for get, True for update/delete).
- **`query`** — Mock a SOQL query response. Verify `QueryResult` fields are populated correctly (total_size, done, records, next_records_url).
- **`query_all` pagination** — Mock a multi-page response where the first page has `done=False` and `nextRecordsUrl`, then a second page with `done=True`. Verify all records are concatenated.
- **`_handle_error`** — Verify `SalesforceAPIError` is raised with correct status code and error data. Test both JSON and non-JSON error responses.
- **`composite` and `composite_tree`** — Mock composite API responses. Verify payload structure (`allOrNone`, `compositeRequest`).
- **`describe` and `get_limits`** — Mock metadata endpoints.
- **Retry behavior** — Verify that transient failures (e.g., 503) trigger the tenacity retry decorator up to 3 attempts with exponential backoff. Verify permanent failures (e.g., 404) propagate immediately.

### 3. Bulk API Client — HIGH

**File:** `src/api/bulk.py` (279 lines, 0 tests)

Essential for high-volume operations. The CSV conversion logic in particular is easy to test in isolation.

**Tests to add (`tests/test_bulk.py`):**

- **`_records_to_csv`** — Pure function, no mocking needed. Test with multiple records, empty list, and records with special characters (commas, quotes, newlines in field values).
- **`_csv_to_records`** — Inverse of the above. Test round-trip: `_csv_to_records(_records_to_csv(records)) == records`.
- **`_create_job`** — Mock the POST request. Verify payload includes correct `object`, `operation`, `contentType`, and optional `externalIdFieldName`.
- **`_upload_data`** — Verify CSV content type header and PUT method.
- **`_execute_job` lifecycle** — Mock the full sequence (create → upload → close → poll → results) and verify the `BulkJobResult` is constructed correctly.
- **Job abort on error** — Verify that if `_upload_data` or `_close_job` fails, `_abort_job` is called.
- **`_poll_job_status` timeout** — Test that `BulkAPIError` is raised when the timeout is exceeded.
- **`delete` helper** — Verify record IDs are converted to `[{"Id": rid}]` format.

### 4. Salesforce Agent — HIGH

**File:** `src/agents/salesforce_agent.py` (330 lines, 0 tests)

The orchestration layer. Tests here validate that tasks are routed and handled correctly.

**Tests to add (`tests/test_agent.py`):**

- **Task routing (`_process_task`)** — Mock the API clients. Submit tasks with each of the 9 operation types and verify the correct handler is called with the correct arguments.
- **Unknown operation** — Verify that an unrecognized operation type causes the task to be marked as failed.
- **Error handling** — Verify that `SalesforceAPIError` from the client causes `queue.fail()` to be called, not a crash.
- **`_handle_update` data mutation** — This handler pops `id` from `task.data`. Verify it works correctly and doesn't corrupt the original data.
- **`_handle_upsert` data extraction** — Verify `external_id_field` and `external_id` are extracted before passing remaining data to the client.
- **`_bulk_result_to_dict`** — Test conversion of `BulkJobResult` to dictionary, including the `success` flag logic (`records_failed == 0`).
- **`get_stats`** — Mock `client.get_limits()` to return sample data. Also test the graceful fallback when `get_limits()` raises an exception.
- **`submit_task` / `submit_batch`** — Verify delegation to `queue.submit` and `queue.submit_batch`.

### 5. Task Queue Gaps — MEDIUM

**File:** `src/queue/task_queue.py` (existing `tests/test_queue.py` has 7 tests)

The existing tests cover the happy path well but miss important edge cases.

**Tests to add to `tests/test_queue.py`:**

- **Max retries exceeded** — Fail a task 3 times (the default `max_retries`) and verify it reaches `FAILED` status permanently instead of being re-queued.
- **Empty queue** — Call `get_next()` on an empty queue and verify it returns `None`.
- **All 4 priority levels** — Current test only checks LOW vs HIGH. Add a test with all 4 priorities (URGENT, HIGH, NORMAL, LOW) and verify exact ordering.
- **Task data integrity** — Submit a task with complex nested data, retrieve it, and verify the data is preserved exactly.
- **Concurrent access** — Use threading to simulate multiple agents calling `get_next()` simultaneously and verify no task is assigned to two agents.

### 6. Configuration & Entry Points — LOW

**Files:** `src/main.py`, `src/daemon.py`, `src/run_sfdx.py`

These are harder to unit test (CLI, daemon loops), but some targeted tests are worthwhile.

**Tests to add:**

- **Config loading (`main.py`)** — Test YAML parsing with a fixture config file. Verify environment variable overrides work.
- **`daemon.py` task routing** — The daemon has its own `_route_task` logic separate from the agent. Verify it delegates correctly.

---

## Infrastructure Recommendations

1. **Add `pytest-cov` to CI** — The dependency is already in `requirements.txt`. Add `pytest --cov=src --cov-report=term-missing` to the CI workflow to get coverage reports on every PR.

2. **Add a `.coveragerc` file** to exclude non-testable code (e.g., `if __name__ == "__main__"` blocks, daemon sleep loops).

3. **Use `responses` or `unittest.mock`** — For mocking HTTP calls in the API client and auth tests. The `responses` library provides a cleaner API for mocking `requests`.

4. **Set a coverage threshold** — Start with 50% and incrementally raise it as tests are added. Enforce in CI with `--cov-fail-under=50`.

---

## Suggested Implementation Order

| Phase | Module | Estimated Tests | Rationale |
|-------|--------|----------------|-----------|
| 1 | `auth/oauth.py` | ~10 tests | Foundation — everything depends on auth |
| 2 | `api/client.py` | ~12 tests | Core CRUD — most common operations |
| 3 | `api/bulk.py` | ~8 tests | CSV helpers are easy wins; job lifecycle is important |
| 4 | `agents/salesforce_agent.py` | ~10 tests | Orchestration — verifies integration of all pieces |
| 5 | `queue/task_queue.py` (gaps) | ~5 tests | Hardens the one module that already has tests |
| 6 | Entry points | ~3 tests | Config parsing and basic smoke tests |

**Total: ~48 new tests** to bring coverage to a reasonable level (~70-80%).
