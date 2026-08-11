# Improvement plan

The following issues were drafted after a deep read of the server, the
persistent-subprocess execution model, the session/tool layer, configuration,
startup helpers, tests, and the CI/DevOps setup. They cover code safety, code
smell, stability, DevOps, and performance, in priority order.

> **Note:** GitHub Issues are disabled on this repository (`POST .../issues`
> returns `410 Issues has been disabled in this repository`), so these could
> not be filed directly. Re-enable Issues under repo Settings, then paste
> each section below in as its own issue (title, labels, and body are already
> formatted for that).

---

## 1. Concurrent tool calls can starve the `restart_session` recovery path on the shared asyncio thread-pool executor

**Labels:** `bug`

### Summary

`open_session`, `close_session`, `execute_python_code`, and `restart_session` all offload their blocking work via `asyncio.to_thread(...)` (`src/ansys/lumerical/mcp/tools.py:162,191,229,363,456`). `asyncio.to_thread` schedules onto the event loop's **default executor**, a `concurrent.futures.ThreadPoolExecutor` sized `min(32, os.cpu_count() + 4)` unless the application overrides it. Nothing in this server does.

`LumericalPersistentPythonSession.execute()` serializes all subprocess access behind a single `threading.Lock` (`_execution_lock`, `persistent_session.py:232`), and `restart()`'s own `start()` call must **re-acquire that same lock** to run the startup snippet in the fresh subprocess — this is explicitly exercised by `tests/test_persistent_session.py::test_restart_unblocks_wedged_execute_holding_lock`, whose docstring even calls out that "`start()`'s startup-code execute ... must acquire `_execution_lock`".

### The gap

Every code comment and docstring in this codebase treats `restart_session` as the universal, always-available escape hatch for a wedged subprocess (e.g. `tools.py:411-416`, `persistent_session.py:35-40`). That's true for *one* wedged call. It stops being true once concurrent tool calls exceed the executor's worker count:

1. N `execute_python_code`/`open_session` calls arrive concurrently (N ≥ executor `max_workers`, e.g. as low as 6 workers on a 2-vCPU box: `min(32, 2+4)`).
2. One snippet wedges (infinite loop, stuck license handshake). Every other in-flight call blocks inside `with self._execution_lock:` waiting for the lock — each burning one executor worker thread for the entire wait.
3. Once all workers are occupied this way, the agent's `restart_session` call is itself submitted via `asyncio.to_thread(py.restart)` (`tools.py:456`) onto the *same, now-saturated* executor. It queues behind the blocked workers and never gets a thread to run on — so the one tool documented to "always" recover a wedged session can't execute at all.

This is a latent risk rather than something observed in the wild here, but the codebase's own design intent (docstrings repeatedly instruct the agent to "dispatch a parallel `restart_session` tool call") assumes concurrent calls are a supported usage pattern — which is exactly the condition that breaks it.

### Suggested fix

Run the blocking Lumerical subprocess calls on a small **dedicated** `ThreadPoolExecutor` owned by the server, and submit `restart_session`'s work to a separate, single-purpose executor/thread so a restart is never stuck behind a queue of calls it exists to unstick:

```python
_LUMERICAL_EXECUTOR = ThreadPoolExecutor(max_workers=8, thread_name_prefix="pylumerical-exec")
_RESTART_EXECUTOR = ThreadPoolExecutor(max_workers=1, thread_name_prefix="pylumerical-restart")

raw = await loop.run_in_executor(_LUMERICAL_EXECUTOR, session.execute, snippet)
...
restart_result = await loop.run_in_executor(_RESTART_EXECUTOR, py.restart)
```

### Acceptance criteria

- [ ] Blocking Lumerical subprocess calls no longer share the asyncio default executor with the rest of the process (or the default executor is explicitly sized/reserved for this purpose).
- [ ] `restart_session` is guaranteed a worker thread regardless of how many `execute_python_code`/`open_session`/`close_session` calls are queued.
- [ ] A regression test reproduces the starvation with an artificially small executor (e.g. `max_workers=1`) and shows `restart_session` still completes.

### Relevant files

- `src/ansys/lumerical/mcp/tools.py` (lines 162, 191, 229, 363, 456)
- `src/ansys/lumerical/mcp/persistent_session.py` (lines 210-368, esp. 232 `_execution_lock`)
- `tests/test_persistent_session.py::test_restart_unblocks_wedged_execute_holding_lock`

---

## 2. Document the execute_python_code trust boundary and harden guidance for non-stdio transports

**Labels:** `documentation`, `enhancement`

### Summary

`execute_python_code` (`src/ansys/lumerical/mcp/tools.py:286-399`) is, by design, an **unsandboxed arbitrary Python code execution** tool: whatever the connected LLM client sends is `exec`'d in a real subprocess with the same OS-level privileges as the server process (file system, network, subprocess spawning — anything Python can do). That's the right design for this product (the assistant needs to drive lumapi freely), but it means the MCP server's effective trust boundary is "whoever can call this tool has a shell."

Today, none of the docs make that boundary explicit:

- `SECURITY.md` covers vulnerability *reporting* but says nothing about the tool's inherent code-execution surface or safe deployment guidance.
- `.env.example` and `doc/source/getting_started/quick_start.rst` show how to switch the transport to `FASTMCP_TRANSPORT=streamable-http` with a host/port, but neither mentions authentication, and there's no auth/allowlist middleware wired into `PyLumericalMCP`/`FastMCP` in `server.py`.
- There's no guidance against binding `FASTMCP_HOST` to `0.0.0.0`/a non-loopback interface, which would expose an unauthenticated arbitrary-code-execution endpoint to the network.

### Why it matters

This server is presumably mostly run locally via stdio today (low risk), but `.env.example`/docs already document the HTTP transport path, so someone will eventually put this behind a real network port. Prompt injection is also a realistic vector for MCP tool servers generally: if the connected LLM is ever fed untrusted content (a shared project file, a web page it was asked to summarize) that content could steer the model into calling `execute_python_code` with malicious code, and the server has no independent guard against that.

### Suggested fix

1. Add a "Security considerations" section to `SECURITY.md` or `README.md` stating plainly: `execute_python_code` provides no sandboxing; only run this server with clients/models you trust, and treat it like giving that client a shell on this machine.
2. Add an explicit warning next to the `streamable-http` instructions in `doc/source/getting_started/quick_start.rst` and `.env.example`: bind to `127.0.0.1` unless a reverse proxy / auth layer terminates in front, and never expose the port to an untrusted network.
3. Investigate whether FastMCP's HTTP transport supports bearer-token / API-key auth (it does, via `fastmcp.server.auth`), and document how to turn it on for this server, or wire up a minimal opt-in token check via `Config`/env var if not already reachable through `PyAnsysBaseMCP`.

### Acceptance criteria

- [ ] SECURITY.md/README documents the trust boundary of `execute_python_code`.
- [ ] Quick-start docs warn against binding the HTTP transport to a public interface without auth.
- [ ] A documented (or implemented) path exists for adding authentication when using `streamable-http`.

### Relevant files

- `src/ansys/lumerical/mcp/tools.py` (lines 286-399)
- `SECURITY.md`
- `.env.example`
- `doc/source/getting_started/quick_start.rst`

---

## 3. open_session re-reads .env/environment on every call instead of reusing the server's resolved Config

**Labels:** `enhancement`

### Summary

`open_session` calls `load_config()` fresh on every invocation to compute `effective_hide`:

```python
# src/ansys/lumerical/mcp/tools.py:146-148
lifespan_context = _lifespan_context(ctx)
cfg = load_config()
effective_hide = cfg.hide_gui if hide is None else bool(hide)
```

`load_config()` (`src/ansys/lumerical/mcp/config.py:49-56`) calls `load_dotenv()`, which re-opens and re-parses the `.env` file from disk, and re-reads every relevant `os.environ` entry — on every single `open_session` call.

Meanwhile, `server.py` already resolves a `Config` once at import time (`config = load_config()`, `server.py:184`) and threads it through `PyLumericalMCP.__init__`/`self._config`, specifically so `install_dir`/`license_file` are fixed at startup.

### Why it matters

- **Wasted I/O**: a disk read + re-parse on every `open_session` call for a value (`LUMERICAL_HIDE_GUI`) that is effectively static for the life of the process.
- **Drift risk**: `python-dotenv`'s `load_dotenv()` does not override already-set process environment variables, so if `.env` is edited while the server is running, `open_session`'s `cfg.hide_gui` can silently diverge from whatever `Config` the rest of the server (e.g. `install_dir`) was built from at startup — two different "current config" views coexisting in the same process for no functional reason.
- **Testability**: any test exercising `open_session`'s hide-default behavior has to patch environment/`.env` state instead of just injecting a `Config`.

### Suggested fix

Resolve `Config` once (already done in `server.py`) and make it reachable from the tool layer — e.g. store it on `PyLumericalContext` or read it off `app._config` — instead of calling `load_config()` again inside `open_session`. If per-call environment overrides are genuinely wanted, that should be an explicit, documented feature, not an accidental side effect of calling `load_config()` a second time.

### Acceptance criteria

- [ ] `open_session` no longer calls `load_config()`/`load_dotenv()` per invocation.
- [ ] The `hide_gui` default is read from the same `Config` instance the rest of the server uses.

### Relevant files

- `src/ansys/lumerical/mcp/tools.py` (lines 146-148)
- `src/ansys/lumerical/mcp/config.py`
- `src/ansys/lumerical/mcp/server.py` (lines 83-90, 184-190)

---

## 4. Split the 1500+ line guideline strings in contexts/fdtd.py out of Python source

**Labels:** `maintenance`, `enhancement`

### Summary

The `get_guidelines_for` topic content lives as giant triple-quoted Python string literals returned from one function per topic:

- `contexts/fdtd.py`: 1542 lines, 9 functions, each returning a multi-hundred-line markdown string (e.g. `get_guidelines_for_fdtd_workflow_example` alone spans ~220 lines).
- `contexts/mode.py`: 739 lines, 4 functions.
- `contexts/interconnect.py`, `contexts/device.py`: similar shape at smaller scale.

### Why it's a smell worth fixing

- **Diff/review noise**: any wording tweak to one guideline topic produces a diff inside a 1500-line Python file, and `git blame`/PR review has to wade through Python-string escaping instead of clean markdown.
- **No content-only ownership**: technical writers or domain experts who want to correct a piece of Lumerical guidance (e.g. a stale lumapi property name) have to touch `.py` files and understand the surrounding function/`__all__` wiring, even though the content is pure markdown with zero logic.
- **Invisible to prose tooling**: the repo already has `doc/styles/typos.toml` and a Vale config (`doc/.vale.ini`) for prose style checks elsewhere in the docs, but this is the largest block of prose in the whole package and it's invisible to those tools because it's embedded in `.py` strings rather than `.md` files.
- **Merge-conflict hotspot**: multiple contributors editing different topics in the same 1500-line file collide more than they would with one file per topic.

### Suggested fix

Move each topic's markdown body into its own `.md` file under e.g. `src/ansys/lumerical/mcp/contexts/data/fdtd/<topic>.md`, and load it via `importlib.resources` — the same mechanism `startup_code.py` already uses to read `_subprocess_helpers.py` as text — e.g.:

```python
def get_guidelines_for_fdtd_workflow() -> str:
    return files("ansys.lumerical.mcp.contexts.data.fdtd").joinpath("workflow.md").read_text()
```

This keeps the public `get_guidelines_for_*` function surface and `__all__` exports identical (no API change), just relocates the content. As a bonus, the resulting `.md` files can be picked up by the existing Vale/typos doc-style tooling.

### Acceptance criteria

- [ ] Guideline prose lives in standalone markdown files, not Python string literals.
- [ ] `get_guidelines_for_*` function signatures/behavior are unchanged (existing tests pass unmodified).
- [ ] New markdown files are covered by the existing doc-style checks (`doc-style` CI job / typos / Vale) where feasible.

### Relevant files

- `src/ansys/lumerical/mcp/contexts/fdtd.py` (1542 lines)
- `src/ansys/lumerical/mcp/contexts/mode.py` (739 lines)
- `src/ansys/lumerical/mcp/contexts/interconnect.py`, `contexts/device.py`
- `src/ansys/lumerical/mcp/startup_code.py` (existing `importlib.resources` pattern to mirror)

---

## 5. pytest-xdist is declared but unused; sleep-heavy persistent-session tests inflate CI wall time across the 8-way matrix

**Labels:** `testing`, `maintenance`

### Summary

`pyproject.toml` declares `pytest-xdist==3.8.0` as a test dependency (line 59), but nothing enables it: `[tool.pytest.ini_options].addopts` has no `-n`/`--numprocesses` flag, and the CI `test` job just calls the shared `ansys/actions/tests-pytest` action with `pytest-markers: '-m "not integration"'` (`.github/workflows/ci.yml:130-133`) — no xdist flag there either. So the dependency currently buys nothing.

Meanwhile `tests/test_persistent_session.py` has several tests that burn real wall-clock time by design (they're testing timing-sensitive recovery behavior, so this is legitimate — flagging the aggregate cost, not the tests themselves):

- `time.sleep(1.5)` (line 79)
- Two subprocess-hang scenarios with `time.sleep(60)`/`time.sleep(120)` inside the child, each preceded by a `time.sleep(0.3)`/`time.sleep(0.5)` on the test thread and followed by `t.join(timeout=8)` / `rt.join(timeout=30)` / `t.join(timeout=10)` (lines 182-309).

None of that is wrong in isolation, but the `test` job runs this file serially across an **8-way matrix** (`ubuntu-latest`/`windows-latest` × Python 3.11-3.14, `ci.yml:118-124`), on top of the also-serial `smoke-tests` matrix that gates it. The declared-but-dormant `pytest-xdist` dependency suggests parallelization was intended at some point and simply never wired up.

### Suggested fix

- Either wire up `pytest-xdist` (`-n auto` in `addopts`, or via whatever parameter the `tests-pytest` action exposes for extra pytest args) so the existing test suite actually parallelizes, or remove the unused dependency if xdist was superseded by something else and simply forgotten.
- Independently, consider whether the `time.sleep(60)`/`time.sleep(120)` "hang" durations in `test_persistent_session.py` need to be that long — the tests already kill/interrupt well before the sleep completes (0.3s/0.5s in), so the sleep duration itself mostly just needs to safely exceed the test's own timeouts, not be realistic.

### Acceptance criteria

- [ ] `pytest-xdist` is either actively used (CI shows parallel worker output) or removed from `[project.optional-dependencies].tests`.
- [ ] No regression in flake rate from parallelization (shared subprocess/thread tests in `test_persistent_session.py` and `test_server.py` should be checked for cross-worker interference before enabling `-n auto` broadly).

### Relevant files

- `pyproject.toml` (lines ~59, ~106-120)
- `.github/workflows/ci.yml` (lines 118-133)
- `tests/test_persistent_session.py`

---

## 6. Pin devcontainer base image and tool versions for reproducible builds

**Labels:** `maintenance`

### Summary

`.devcontainer/Dockerfile` builds from a floating tag and installs a tool without a version pin:

```dockerfile
FROM --platform=linux/amd64 rockylinux:9
...
RUN pip install uv
```

`rockylinux:9` tracks the latest point release of RHEL 9 (it is regularly repointed to new digests upstream), and `pip install uv` always grabs whatever the latest `uv` release is at build time. The rest of the project is careful about pinning (GitHub Actions in `.github/workflows/ci.yml` are all pinned to full commit SHAs with version comments; `pyproject.toml` test/doc extras pin exact versions), so this is an inconsistency rather than a considered choice.

### Why it matters

- A devcontainer rebuilt today vs. next month can silently pick up a different base OS point release or a new major `uv` version, producing "works on my machine" drift between contributors and making build failures harder to bisect (was it my change, or did the base image move under me?).
- `uv` is under active development; an unpinned install risks breaking `uv sync --extra tests` / `uv tool install --force pre-commit` in `post-create.sh` on a future breaking release.

### Suggested fix

- Pin `FROM rockylinux:9` to a digest (`rockylinux:9@sha256:...`) or at least a more specific tag if Rocky publishes them, and document the update process (or track the digest with Dependabot/Renovate the way `dependabot.yml` already tracks pip/Actions).
- Pin `uv` to a specific version in the `pip install uv` step, matching the precision already used elsewhere in the repo.

### Acceptance criteria

- [ ] Base image reference includes a digest (or otherwise reproducible pin).
- [ ] `uv` install is version-pinned.
- [ ] (Optional) Dependabot is configured to track these pins going forward, consistent with the existing `pip`/`github-actions` ecosystems in `.github/dependabot.yml`.

### Relevant files

- `.devcontainer/Dockerfile`
- `.github/dependabot.yml` (for context on the existing pinning conventions)

---

## 7. Re-enable Codecov patch coverage so new/changed code is covered by a gate

**Labels:** `testing`, `maintenance`

### Summary

`codecov.yml` currently disables the patch coverage check entirely:

```yaml
coverage:
  range: 80..100
  round: down
  precision: 2
  status:
    project:
      default:
        target: 80%
    patch: off
```

Only the *project-wide* 80% target is enforced; `patch: off` means a PR that adds a large chunk of entirely uncovered code can still pass Codecov as long as the overall project percentage doesn't dip below 80% (easy to stay under, given the size of the existing well-tested suite).

### Why it matters

Patch coverage is usually the more actionable signal for reviewers: it flags "this PR's own new lines aren't tested" right where the diff is, rather than relying on the aggregate project number to eventually catch it. With `patch: off`, a regression in test discipline on new code isn't caught by CI at all — only project-level erosion over many PRs would eventually trip the 80% floor, by which point the specific untested lines are harder to trace back to a single PR.

### Suggested fix

Turn patch coverage back on with a threshold that fits the maintainers' actual review bar, e.g.:

```yaml
coverage:
  status:
    project:
      default:
        target: 80%
    patch:
      default:
        target: 70%
        threshold: 5%
```

(numbers illustrative — tune to whatever bar makes sense for new code, possibly lower than the project target initially and ratcheted up over time).

### Acceptance criteria

- [ ] `codecov.yml` enforces a patch coverage threshold instead of `patch: off`.
- [ ] A follow-up PR with intentionally-uncovered new lines demonstrates the check now fails as expected (can be verified in a draft PR, then reverted).

### Relevant files

- `codecov.yml`
