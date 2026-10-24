# Installation, Uninstallation & CLI Reference (`INSTALL.md`)

This guide covers every supported way to install, verify, use, and uninstall `ci-proof`, along with a complete reference for every CLI subcommand and flag.

---

## 1. Compatibility & System Requirements

| Requirement | Supported | Notes |
|---|---|---|
| **Operating Systems** | Linux (`x86_64`, `aarch64`), macOS (`arm64`, `x86_64`), Windows via **WSL2** | Native Windows (`cmd.exe` / PowerShell) is not supported by `scripts/install.sh`; use WSL2 (Ubuntu 22.04 or 24.04) or Docker on Windows. |
| **Python Versions** | Python `3.9`, `3.10`, `3.11`, `3.12`, `3.13` | Core engine uses only the Python standard library (`dependencies = []`). |
| **YAML Profile Parsing** | `PyYAML >= 6.0` (optional) or system `ruby` | Compiled `.idx` indexes embed the parsed profile and require neither. To parse `.yaml` profiles directly on minimal Linux systems without Ruby, install with `pip install ".[yaml]"`. |
| **Network Egress** | **Zero** for `index`, `select`, `affected-tests`, `verify`, `report` | Only `audit`, `flaky`, and `shadow` make read-only HTTPS requests to CI provider APIs (GitHub, GitLab, Prow/GCS). |

---

## 2. Installation Methods

### Method 1: Standalone Local Installer (`scripts/install.sh`)

Installs a self-contained copy of the engine and built-in profiles under `~/.ci-proof` and symlinks the `ci-proof` executable into `~/.local/bin/ci-proof` without modifying your system Python packages.

> **Status note:** Remote `curl` installation from a release tarball is **Planned (Stage D1)** once the repository is made public. Until D1, run `scripts/install.sh` from a local clone.

#### Install
```bash
bash scripts/install.sh
```

You can customize the target directories with environment variables:
```bash
CI_PROOF_INSTALL_DIR="$HOME/.ci-proof" CI_PROOF_BIN_DIR="$HOME/.local/bin" bash scripts/install.sh
```

#### What Gets Installed Where
| Path | Type | Purpose |
|---|---|---|
| `~/.ci-proof/lib/engine/` | Directory | Copy of the `engine/` Python package (bytecode-free) |
| `~/.ci-proof/lib/profiles/` | Directory | Built-in repository profiles (`*.yaml`) and `schema.json` |
| `~/.ci-proof/bin/ci-proof` | Executable script | Symlink-safe launcher that sets `PYTHONPATH` and invokes `engine.cli` |
| `~/.local/bin/ci-proof` | Symbolic link | Points to `~/.ci-proof/bin/ci-proof` |

#### Verify
```bash
~/.local/bin/ci-proof --version
# Output: ci-proof 0.1.0
```

#### Uninstall (`scripts/uninstall.sh`)
```bash
bash scripts/uninstall.sh
```
`scripts/uninstall.sh` removes `~/.local/bin/ci-proof` and `~/.ci-proof` (and cleans up `~/.local/bin` / `~/.local` if they are empty).

**Zero-residue verification in a clean sandbox (`HOME`):**
```bash
TMP_HOME="$(mktemp -d)"
HOME="$TMP_HOME" bash scripts/install.sh
HOME="$TMP_HOME" "$TMP_HOME/.local/bin/ci-proof" --version
HOME="$TMP_HOME" bash scripts/uninstall.sh
test -z "$(find "$TMP_HOME" -mindepth 1)" && echo "PASS: 0 files remaining"
rmdir "$TMP_HOME"
```

---

### Method 2: Python Package (`pip` in a Virtualenv or `pipx`)

#### Option A: `pip` (Virtual Environment)
```bash
python3 -m venv .venv
.venv/bin/pip install ".[yaml]"
.venv/bin/ci-proof --version
```
- **What gets installed where:** `<venv>/lib/python3.X/site-packages/engine/`, `<venv>/lib/python3.X/site-packages/profiles/` (including `schema.json` and `*.yaml`), and the console entry point `<venv>/bin/ci-proof`.
- **Uninstall:**
  ```bash
  .venv/bin/pip uninstall -y ci-proof pyyaml
  rm -rf .venv
  ```

#### Option B: `pipx` (Isolated User-Level CLI)
```bash
pipx install ".[yaml]"
ci-proof --version
```
- **What gets installed where:** Isolated virtual environment in `~/.local/pipx/venvs/ci-proof/` and binary symlink at `~/.local/bin/ci-proof`.
- **Uninstall:**
  ```bash
  pipx uninstall ci-proof
  ```

---

### Method 3: Docker Container (Local Build)

> **Status note:** No pre-built image is published to `ghcr.io` yet (planned for public release D1). Build the image locally from the repository root.

#### Build & Smoke Test
```bash
docker build -t ci-proof:local .
docker run --rm ci-proof:local --version
# Output: ci-proof 0.1.0
```

#### Run Against a Local Workspace
Mount your repository or working directory at `/workspace`:
```bash
docker run --rm -v "$(pwd):/workspace" ci-proof:local --help
```

- **What gets installed inside the image:** `/opt/ci-proof/engine/`, `/opt/ci-proof/profiles/`, and `/usr/local/bin/ci-proof` on `python:3.11-alpine` with `git`, `bash`, and `pyyaml`.
- **Uninstall (remove local image):**
  ```bash
  docker rmi ci-proof:local
  ```

---

### Method 4: GitHub Action (`action.yml`)

> **Status note:** Because this repository is currently **private** (pre-D1), external repositories cannot reference `uses: Aamir-raza-1/ci-casestudies@main`.
> - Workflows **inside this repository** reference `uses: ./`.
> - Workflows in **external repositories or public forks** must vendor `action.yml`, `pyproject.toml`, `setup.py`, `README.md`, `engine/`, and `profiles/` into `.github/actions/ci-proof/` and reference `uses: ./.github/actions/ci-proof`.

#### Example Workflow Step
```yaml
- name: Change-Aware Test Selection
  id: select
  uses: ./.github/actions/ci-proof
  with:
    profile: profiles/home-assistant__core.yaml
    index: ha.idx
    command: pytest
```
- **Uninstall:** Remove the workflow step (and `.github/actions/ci-proof/` if vendored).

---

### Method 5: Model Context Protocol (MCP) Server (`ci-proof mcp`)

Runs a JSON-RPC 2.0 server over `stdio` (MCP specification `2024-11-05`) for AI coding assistants (Claude Code, Cursor, Windsurf, VS Code Copilot, Antigravity).

#### Configure
First install `ci-proof` via Method 1 (`scripts/install.sh`) or Method 2 (`pipx install ".[yaml]"`), then add to your MCP client config:
```json
{
  "mcpServers": {
    "ci-proof": {
      "command": "ci-proof",
      "args": ["mcp"]
    }
  }
}
```

#### Smoke Test Over Stdio
```bash
python3 -c '
import json, subprocess
p = subprocess.Popen(["ci-proof", "mcp"], stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True)
p.stdin.write(json.dumps({"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {}}) + "\n")
p.stdin.flush()
resp = json.loads(p.stdout.readline())
assert resp["result"]["serverInfo"]["name"] == "ci-proof", resp
print("MCP initialized:", resp["result"]["serverInfo"])
p.terminate()
'
```
- **Uninstall:** Remove the `"ci-proof"` block from `mcpServers` in your client configuration file.

---

## 3. Complete CLI Reference & Runnable Examples

### Global Flags

| Flag | Description | Example |
|---|---|---|
| `--version` | Print `ci-proof <version>` and exit | `ci-proof --version` |
| `-h`, `--help` | Show command overview or subcommand help | `ci-proof --help` |

---

### 3.1 `ci-proof index` — Compile Static + Runtime Maps Into a Binary Index

Compiles a static import map (`{test_file: [repo_files]}`) and optional runtime file-open/coverage map into a single compact `.idx` file (`CIDX1` format) with embedded profile metadata.

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Path to per-repo profile (`.yaml` or `.json`) |
| `--static PATH` | Yes | Static import map JSON (`{test_file: [repo_files]}`) |
| `--map PATH` | No | Runtime map from traced run (`map_testfiles.json.gz` or `.json`) |
| `--map-commit SHA` | No | Git commit SHA at which the runtime map was traced |
| `--map-date TEXT` | No | Timestamp when the runtime map was traced (defaults to file mtime) |
| `--out PATH` | Yes | Output path for the compiled index (e.g. `repo.idx`) |

**Runnable example (works on any fresh clone):**
```bash
TMP_DIR="$(mktemp -d)"
printf '{"tests/components/hue/test_light.py":["homeassistant/components/hue/light.py"],"tests/components/zwave/test_init.py":["homeassistant/components/zwave/init.py"]}\n' > "$TMP_DIR/static.json"
printf '{"test_files":["tests/components/hue/test_light.py"],"file_to_test_files":{"repo:homeassistant/components/hue/light.py":[0]}}\n' | gzip > "$TMP_DIR/map.json.gz"

ci-proof index \
  --profile profiles/home-assistant__core.yaml \
  --static "$TMP_DIR/static.json" \
  --map "$TMP_DIR/map.json.gz" \
  --map-commit abc1234 \
  --map-date 2026-10-24T00:00:00Z \
  --out "$TMP_DIR/demo.idx"

rm -rf "$TMP_DIR"
```

---

### 3.2 `ci-proof affected-tests` — List Test Targets Affected by Changed Files

Outputs the sorted list of affected test files (one per line), and optionally writes typed per-test decisions (`MUST_RUN`, `SAFE_TO_SKIP`, `UNCERTAIN`) with proof footprints to a JSON file.

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Path to per-repo profile (`.yaml` or `.json`) |
| `--index PATH` | Yes (or `--static`) | Compiled `.idx` file from `ci-proof index` |
| `--static PATH` | Yes (if no `--index`) | Static import map JSON |
| `--map PATH` | No | Runtime map (`.json.gz` or `.json`, used with `--static`) |
| `--map-commit SHA` | No | Commit SHA recorded in decision proofs (used with `--static`) |
| `--changed FILE [FILE ...]` | Yes | Changed repository file paths (treated as module-level edits unless listed in `--body-only`) |
| `--body-only FILE [FILE ...]` | No | Subset of changed files whose edits are inside function bodies only |
| `--explain` | No | Write typed decisions (`engine.decisions/v1`) to `--out` |
| `--out PATH` | With `--explain` | Output JSON path for typed decisions |

**Runnable example (works on any fresh clone):**
```bash
TMP_DIR="$(mktemp -d)"
printf '{"tests/components/hue/test_light.py":["homeassistant/components/hue/light.py"],"tests/components/zwave/test_init.py":["homeassistant/components/zwave/init.py"]}\n' > "$TMP_DIR/static.json"
printf '{"test_files":["tests/components/hue/test_light.py"],"file_to_test_files":{"repo:homeassistant/components/hue/light.py":[0]}}\n' | gzip > "$TMP_DIR/map.json.gz"
ci-proof index --profile profiles/home-assistant__core.yaml --static "$TMP_DIR/static.json" --map "$TMP_DIR/map.json.gz" --map-commit abc1234 --out "$TMP_DIR/demo.idx"

ci-proof affected-tests \
  --profile profiles/home-assistant__core.yaml \
  --index "$TMP_DIR/demo.idx" \
  --changed homeassistant/components/hue/light.py \
  --body-only homeassistant/components/hue/light.py \
  --explain \
  --out "$TMP_DIR/decisions.json"

rm -rf "$TMP_DIR"
```

---

### 3.3 `ci-proof select` — Single-Diff Summary or Historical PR Replay

Evaluates either a single list of changed files (printing count + percentage share + rule reasons) or replays a batch of historical PR diffs to compute time-weighted CI savings.

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Path to per-repo profile (`.yaml` or `.json`) |
| `--index PATH` | Yes (or `--static`) | Compiled `.idx` index |
| `--static PATH` | Yes (if no `--index`) | Static import map JSON |
| `--map PATH` | No | Runtime map (`.json.gz` or `.json`, used with `--static`) |
| `--map-commit SHA` | No | Commit SHA for runtime map provenance |
| `--changed FILE [FILE ...]` | Mutually exclusive with `--replay` | Changed files to evaluate |
| `--replay PATH` | Mutually exclusive with `--changed` | JSON file with `{"prs": [{"pr": 1, "minutes": 20, "files": [...]}]}` |
| `--seconds PATH` | No | JSON file mapping `{test_file: duration_seconds}` for time-weighted shares |
| `--out PATH` | No | Output JSON path for full replay results (`summary` + `prs`) |

**Runnable example (works on any fresh clone):**
```bash
TMP_DIR="$(mktemp -d)"
printf '{"tests/components/hue/test_light.py":["homeassistant/components/hue/light.py"],"tests/components/zwave/test_init.py":["homeassistant/components/zwave/init.py"]}\n' > "$TMP_DIR/static.json"
printf '{"tests/components/hue/test_light.py":10.0,"tests/components/zwave/test_init.py":30.0}\n' > "$TMP_DIR/seconds.json"
printf '{"prs":[{"pr":101,"minutes":40.0,"files":["homeassistant/components/hue/light.py"]}]}\n' > "$TMP_DIR/replay.json"

# Single change summary:
ci-proof select \
  --profile profiles/home-assistant__core.yaml \
  --static "$TMP_DIR/static.json" \
  --seconds "$TMP_DIR/seconds.json" \
  --changed homeassistant/components/hue/light.py

# Replay summary:
ci-proof select \
  --profile profiles/home-assistant__core.yaml \
  --static "$TMP_DIR/static.json" \
  --seconds "$TMP_DIR/seconds.json" \
  --replay "$TMP_DIR/replay.json" \
  --out "$TMP_DIR/replay_out.json"

rm -rf "$TMP_DIR"
```

---

### 3.4 `ci-proof verify` — Safety Verification on Planted Bugs (Mutation + Isolation)

Checks whether test selection catches every planted bug killed by the full test suite, separating baseline failures, flaky tests, and order-dependent failures via isolated re-runs.

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Path to per-repo profile (`.yaml` or `.json`) |
| `--junit DIR` | Yes | Directory containing `junit-b<batch>-g<group>.xml` (`batch 0` = baseline) |
| `--mutation DIR` | Yes | Directory containing `mutants.json` and `batches.json` |
| `--static PATH` | Yes | Static import map JSON |
| `--runtime PATH` | Yes | Runtime map (`map_testfiles.json.gz`) |
| `--isolate DIR` | No | Directory with `junit-m<id>-r<k>.xml` isolated re-runs (`id 0` = clean baseline) |
| `--out PATH` | No | Output JSON path for safety summary and per-mutant verdicts |

**Runnable example (works on any fresh clone):**
```bash
TMP_DIR="$(mktemp -d)"
mkdir -p "$TMP_DIR/junit" "$TMP_DIR/mutation" "$TMP_DIR/isolate"
printf '[{"id":1,"file":"homeassistant/components/hue/light.py","kind":"body","op":"ret","line":10}]\n' > "$TMP_DIR/mutation/mutants.json"
printf '[[],[1]]\n' > "$TMP_DIR/mutation/batches.json"
printf '{"tests/components/hue/test_light.py":["homeassistant/components/hue/light.py"]}\n' > "$TMP_DIR/static.json"
printf '{"test_files":["tests/components/hue/test_light.py"],"file_to_test_files":{"repo:homeassistant/components/hue/light.py":[0]}}\n' | gzip > "$TMP_DIR/rt.gz"
printf '<testsuites><testsuite><testcase classname="c" name="t1" file="tests/components/hue/test_light.py"/></testsuite></testsuites>\n' > "$TMP_DIR/junit/junit-b0-g1.xml"
printf '<testsuites><testsuite><testcase classname="c" name="t1" file="tests/components/hue/test_light.py"><failure/></testcase></testsuite></testsuites>\n' > "$TMP_DIR/junit/junit-b1-g1.xml"
printf '<testsuites><testsuite><testcase classname="c" name="t1" file="tests/components/hue/test_light.py"/></testsuite></testsuites>\n' > "$TMP_DIR/isolate/junit-m0-r1.xml"
printf '<testsuites><testsuite><testcase classname="c" name="t1" file="tests/components/hue/test_light.py"><failure/></testcase></testsuite></testsuites>\n' > "$TMP_DIR/isolate/junit-m1-r1.xml"

ci-proof verify \
  --profile profiles/home-assistant__core.yaml \
  --junit "$TMP_DIR/junit" \
  --mutation "$TMP_DIR/mutation" \
  --static "$TMP_DIR/static.json" \
  --runtime "$TMP_DIR/rt.gz" \
  --isolate "$TMP_DIR/isolate" \
  --out "$TMP_DIR/safety.json"

rm -rf "$TMP_DIR"
```

---

### 3.5 `ci-proof report` — Generate Comparative Audit Report (Markdown + Self-Contained HTML)

Combines `audit`, `flaky`, and optional `selection` and `safety` JSON outputs into an executive waste report with labelled footnotes (`measured`, `extrapolated`, `estimate`, `assumption`) and a self-contained `.html` file.

| Option | Required | Description |
|---|---|---|
| `--audit PATH` | Yes | Summary JSON from `ci-proof audit` |
| `--flaky PATH` | Yes | Flaky report JSON from `ci-proof flaky` |
| `--selection PATH` | No | Replay result JSON from `ci-proof select --replay` |
| `--safety PATH` | No | Safety result JSON from `ci-proof verify` |
| `--rates PATH` | No | YAML/JSON overriding default runner $/min and developer $/hr rates |
| `--out PATH` | Yes | Output Markdown path (`.html` is written alongside it) |

**Runnable example (works on any fresh clone using `tools/fixtures/report_inputs.json`):**
```bash
TMP_DIR="$(mktemp -d)"
python3 -c '
import json, sys
fx = json.load(open("tools/fixtures/report_inputs.json"))
for k in ("audit", "flaky", "selection", "safety", "rates"):
    json.dump(fx[k], open(f"{sys.argv[1]}/{k}.json", "w"))
' "$TMP_DIR"

ci-proof report \
  --audit "$TMP_DIR/audit.json" \
  --flaky "$TMP_DIR/flaky.json" \
  --selection "$TMP_DIR/selection.json" \
  --safety "$TMP_DIR/safety.json" \
  --rates "$TMP_DIR/rates.json" \
  --out "$TMP_DIR/report.md"

ls -la "$TMP_DIR/report.md" "$TMP_DIR/report.html"
rm -rf "$TMP_DIR"
```

---

### 3.6 `ci-proof audit` — Read-Only CI History Audit

Fetches and summarizes workflow runs and job durations over a fixed date window (`[end - days, end)`).

| Option | Required | Description |
|---|---|---|
| `--provider {github,gitlab,prow}` | Yes | CI provider platform |
| `--project ID` | Yes | Repository identifier (e.g. `python/mypy`, `gitlab-org/gitlab`, `kubernetes/kubernetes`) |
| `--days N` | Yes | Number of full days before `--end` |
| `--end YYYY-MM-DD` | Yes | Exclusive UTC end date (fixed window for reproducibility) |
| `--test-job-regex REGEX` | No | Regex matching test job names |
| `--full-run-rule REGEX` | No | Regex marking a workflow run as full-suite |
| `--workflow FILE` | No | Limit to specific workflow file (GitHub, repeatable) |
| `--job NAME` | No | Prow job name (required when `--provider prow`) |
| `--out PATH` | No | Output JSON path (default `experiments/_audits/<provider>__<project>.json`) |

**Example:**
```bash
ci-proof audit \
  --provider github \
  --project python/mypy \
  --days 7 \
  --end 2026-10-10 \
  --workflow test.yml \
  --test-job-regex "(?i)pytest|test" \
  --full-run-rule "(?i)pytest" \
  --out /tmp/mypy_audit.json
```

---

### 3.7 `ci-proof flaky` — Flaky Run & Test Detector

Detects fail-then-pass episodes on the same `(workflow, commit)`, parses failed test logs, classifies failure categories, and estimates wasted runner and developer wait minutes.

| Option | Required | Description |
|---|---|---|
| `--provider {github,gitlab,prow}` | Yes | CI provider (currently implemented for `github`) |
| `--project ID` | Yes | Repository identifier (`owner/repo`) |
| `--days N` | Yes | Number of full days before `--end` |
| `--end YYYY-MM-DD` | Yes | Exclusive UTC end date |
| `--workflow FILE` | No | Limit to workflow file (repeatable) |
| `--event NAME` | No | Filter by trigger event (repeatable; default all PR events) |
| `--test-job-regex REGEX` | No | Regex for test jobs (default `(?i)test`) |
| `--parser {pytest,vitest,jest}` | No | Log parser(s) to apply (repeatable; default `pytest` + `vitest`) |
| `--out PATH` | No | Output JSON path (default `experiments/_audits/flaky__<provider>__<project>.json`) |

**Example:**
```bash
ci-proof flaky \
  --provider github \
  --project pydantic/pydantic \
  --days 7 \
  --end 2026-10-10 \
  --workflow ci.yml \
  --event pull_request \
  --test-job-regex "(?i)test" \
  --parser pytest \
  --out /tmp/pydantic_flaky.json
```

---

### 3.8 `ci-proof shadow` — Passive Shadow Evaluation Against Real CI Failures

Replays selection rules against historical failing CI runs from public logs to measure recall (`caught`, `missed`, `flaky`, `setup-only`).

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Path to per-repo profile |
| `--index PATH` | Yes (or `--static`) | Compiled `.idx` index |
| `--static PATH` | Yes (if no `--index`) | Static import map JSON |
| `--map PATH` | No | Runtime map |
| `--map-commit SHA` | No | Commit SHA for runtime map provenance |
| `--provider NAME` | Yes | Record source (`github` or `prow`) |
| `--project ID` | Yes | Project identifier (`owner/repo`) |
| `--days N` | Yes | Number of full days before `--end` |
| `--end YYYY-MM-DD` | Yes | Exclusive UTC end date |
| `--workflow FILE` | No | Filter by workflow file (repeatable) |
| `--event NAME` | No | Filter by trigger event (repeatable) |
| `--test-job-regex REGEX` | No | Failed jobs whose logs are parsed (default `(?i)test`) |
| `--parser {pytest,vitest,jest}` | No | Log parser(s) (repeatable; default `pytest`) |
| `--out PATH` | Yes | Output path for `shadow.json` |

**Example:**
```bash
ci-proof shadow \
  --profile profiles/openedx__openedx-platform.yaml \
  --index experiments/openedx/openedx.idx \
  --provider github \
  --project openedx/openedx-platform \
  --days 7 \
  --end 2026-10-10 \
  --workflow ci.yml \
  --event pull_request \
  --test-job-regex "(?i)pytest|test" \
  --parser pytest \
  --out /tmp/openedx_shadow.json
```

---

### 3.9 `ci-proof mcp` — Model Context Protocol Stdio Server

Starts the JSON-RPC 2.0 MCP server on `stdin`/`stdout`. Accepts no additional flags (`ci-proof mcp --help`).

**Runnable example:**
```bash
printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}\n' | ci-proof mcp
```

---

### 3.10 `ci-proof trace` — Traced Test Suite Execution (**Planned — Stage R1**)

| Option | Required | Description |
|---|---|---|
| `--profile PATH` | Yes | Validates the profile against `profiles/schema.json` and exits with code `2` (`not implemented yet`) |

> **Status note:** Tracer execution currently runs via `script/trace/` in the repository fork workflows (`trace-tests.yml`, `analyze-traces.yml`). Merging it into `ci-proof trace` is scheduled for checkpoint `R1`.

**Runnable example:**
```bash
ci-proof trace --profile profiles/home-assistant__core.yaml || echo "Exited with status $? (expected 2 until R1)"
```
