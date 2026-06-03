# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run tests (from project root):**
```bash
./test/run
# Equivalent to: python3.9 -m pytest -n auto --flake8 --dist loadfile --ignore=test/integration/modules/ --durations=5 --cov-report html --cov=. .
```

**Run a single test file:**
```bash
python3 -m pytest test/unit/test_spiderfoot.py -v
```

**Run a specific test:**
```bash
python3 -m pytest test/unit/test_spiderfoot.py::SpiderFootTestCase::test_something -v
```

**Integration tests (require third-party API keys — excluded from default runs):**
```bash
python3 -m pytest test/integration/modules/ -v
```

**Acceptance tests (require running web server on port 5001):**
```bash
# Start server first: python3 sf.py -l 127.0.0.1:5001
robot test/acceptance/
```

**Start the web UI:**
```bash
python3 sf.py -l 127.0.0.1:5001
```

**Start CLI mode:**
```bash
python3 sfcli.py -s http://127.0.0.1:5001
```

**Install dependencies:**
```bash
pip install -r requirements.txt
pip install -r test/requirements.txt  # for development
```

## Code Style

- Max line length: **120 characters**
- Max cyclomatic complexity: **60**
- Docstring convention: **Google style**
- Flake8 with plugins enforced: B, DAR, DUO, R, A, S, Q0, SIM, SFS
- Per-file ignores are configured in `setup.cfg` for `plugin.py`, `db.py`, `sf.py`, and all `modules/sfp_*.py`

## Architecture

SpiderFoot is a plugin-based OSINT automation tool. A scan takes a **target** (IP, domain, email, etc.), runs **modules** that emit **events**, stores results in **SQLite**, then optionally runs the **correlation engine** over accumulated results.

### Core Components

**`spiderfoot/` package** — the framework library:
- `plugin.py` — `SpiderFootPlugin` base class all modules inherit from. Implements the publisher/subscriber pattern: modules declare which event types they `watchedEvents()` and produce via `producedEvents()`.
- `event.py` — `SpiderFootEvent`: the unit of data passed between modules. Carries `eventType`, `data`, `confidence`/`visibility`/`risk` (0–100), and a reference to its source event (forming a dependency chain).
- `db.py` — `SpiderFootDb`: SQLite abstraction (thread-safe via `RLock`). Tables: `tbl_scan_instance`, `tbl_scan_results`, `tbl_scan_correlation_results`, `tbl_scan_log`, `tbl_scan_config`, `tbl_event_types`.
- `correlation.py` — `SpiderFootCorrelator`: YAML-based rule engine. Reads rules from `/correlations/`, validates syntax, and runs analysis over stored scan results.
- `helpers.py` — `SpiderFootHelpers`: static utility methods (network parsing, HTML processing, graph generation, module loading, path management).
- `target.py` — `SpiderFootTarget`: validates and tracks the scan target and its aliases. Valid types: `IP_ADDRESS`, `IPV6_ADDRESS`, `INTERNET_NAME`, `EMAILADDR`, `HUMAN_NAME`, `BGP_AS_OWNER`, `PHONE_NUMBER`, `USERNAME`, `BITCOIN_ADDRESS`, etc.
- `threadpool.py` — `SpiderFootThreadPool`: multiprocessing worker pool for parallel module execution.

**Top-level entry points:**
- `sf.py` — Main launcher; `-l host:port` starts the CherryPy web server.
- `sfscan.py` — `SpiderFootScanner`: orchestrates scan lifecycle (QUEUED → RUNNING → COMPLETED/STOPPED/ERROR), manages module instances, processes the event queue.
- `sfwebui.py` — CherryPy web application with REST endpoints, Mako templates, export handlers (CSV, JSON, GEXF, Excel).
- `sfcli.py` — Interactive CLI for remote interaction with a running SpiderFoot server.
- `sflib.py` — Shared utility functions used by the entry points.

**`modules/` directory** — 235 OSINT modules (`sfp_*.py`), plus two special storage modules:
- `sfp__stor_db.py` — writes all events to the database.
- `sfp__stor_stdout.py` — writes events to stdout (CLI mode).

**`correlations/` directory** — 37 YAML correlation rules detecting patterns (malware indicators, exposed databases, open ports, certificate issues, etc.).

### Data Flow

1. User specifies a target and selects modules via web UI or CLI.
2. `SpiderFootScanner` instantiates selected modules and seeds the event queue with the target event.
3. Each module's `handleEvent()` is called when a subscribed event type arrives; modules emit new `SpiderFootEvent` objects.
4. `sfp__stor_db` persists every event to SQLite.
5. After scanning, `SpiderFootCorrelator` runs YAML rules against stored results and writes correlation findings.
6. Web UI or CLI reads results from SQLite for display/export.

### Adding a New Module

Create `modules/sfp_mymodule.py` inheriting from `SpiderFootPlugin`. Implement:
- `describeYourself()` — metadata dict (name, description, category, etc.)
- `watchedEvents()` — list of event types to subscribe to
- `producedEvents()` — list of event types this module emits
- `handleEvent(event)` — processing logic; call `self.notifyListeners(evt)` to emit events

### Testing Patterns

Test files live in `test/unit/`. The `test/conftest.py` provides pytest fixtures:
- `default_options` — standard SpiderFoot config dict for unit tests
- `web_default_options` — config for web UI tests
- `cli_default_options` — config for CLI tests

Module tests typically instantiate the module with `default_options`, mock HTTP responses using the `responses` library, and verify emitted events.

### Environment Variables

- `SPIDERFOOT_DATA` — override data directory path
- `SPIDERFOOT_CACHE` — override cache directory path
- `SPIDERFOOT_LOGS` — override log directory path
