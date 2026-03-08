# laruche

**A registry of installation and testing instructions for 5,000+ PyPI
packages, built for [labeille](https://github.com/devdanzin/labeille)
JIT bug hunting.**

## What is this?

laruche stores YAML files describing how to install and test PyPI packages.
Each file contains the exact `pip install` and `pytest`/`unittest` commands
needed to exercise a package's test suite, along with metadata about
dependencies, quirks, and compatibility.

The registry is consumed by [labeille](https://github.com/devdanzin/labeille),
a tool that hunts for CPython JIT compiler bugs by running real-world test
suites against JIT-enabled Python builds. When labeille finds a crash (SIGABRT,
SIGSEGV) during a test run, it means the JIT compiler produced incorrect code
-- a real bug worth reporting.

You don't need labeille to use laruche. Any tool that needs to know how to
install and test a Python package can read these YAML files directly.

## Quick Start

### Using with labeille

```bash
# Install labeille
pipx install labeille

# Fetch the registry
labeille registry sync

# Run test suites against a JIT Python build
labeille run --target-python /path/to/jit-python --top 50
```

### Using standalone

```bash
# Clone the registry
git clone https://github.com/devdanzin/laruche.git

# Browse a package
cat laruche/packages/click.yaml
```

```yaml
package: click
repo: https://github.com/pallets/click/
pypi_url: https://pypi.org/project/click/
extension_type: pure
python_versions:
- '3.15'
- '3.14'
install_method: pip
install_command: pip install -e . && pip install pytest
test_command: python -m pytest tests/
test_framework: pytest
uses_xdist: false
timeout: null
skip: false
skip_reason: null
skip_versions: {}
notes: Click CLI framework by Pallets. Uses flit_core build backend.
enriched: true
clone_depth: null
import_name: null
```

## Repository Structure

```
laruche/
├── packages/           # 5,000+ per-package YAML files
│   ├── click.yaml
│   ├── requests.yaml
│   ├── flask.yaml
│   └── ...
├── index.yaml          # Summary index sorted by download count
├── migrations.log      # Applied schema migration history
├── schema.yaml         # Schema version for compatibility checking
└── LICENSE             # MIT
```

- `packages/` -- One YAML file per PyPI package, containing install/test
  commands and metadata. File names match PyPI package names.
- `index.yaml` -- Auto-generated summary of all packages, sorted by PyPI
  download count. Never edit manually; rebuild with
  `labeille registry rebuild-index`.
- `migrations.log` -- JSONL log of applied schema migrations (timestamps,
  file counts). Prevents re-application.
- `schema.yaml` -- Contains `schema_version` and `min_labeille_version`.
  labeille checks this at load time to ensure compatibility.

## Package YAML Schema

Each package YAML file contains the fields described below, organized into
five categories.

### Identity fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `package` | `str` | (required) | PyPI package name (must match the filename) |
| `repo` | `str \| None` | `None` | Source repository URL (set by `labeille resolve`) |
| `pypi_url` | `str` | `""` | Link to the PyPI page |
| `import_name` | `str \| None` | `None` | Python import name when it differs from the package name. For example, `Pillow` imports as `PIL`, `python-dateutil` imports as `dateutil`. Used for post-install import validation. Null means the import name is derived from the package name automatically. |

### Classification

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `extension_type` | `str` | `"unknown"` | `pure` (pure Python), `extensions` (has C/Rust extensions), or `unknown`. Set by `labeille resolve` from wheel tag analysis. |
| `clone_depth` | `int \| None` | `None` | Git clone depth. Null means shallow (depth=1). Set to 50 or higher for packages that use setuptools-scm or other version-from-git-tags tools, since shallow clones lose the tags needed for version detection. |

### Installation

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `install_method` | `str` | `"pip"` | `pip`, `pip-extras`, or `custom`. Informational -- the actual command is in `install_command`. |
| `install_command` | `str` | `""` | The exact shell command to install the package with its test dependencies. Runs inside an activated venv with the target Python. |

### Testing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `test_command` | `str` | `""` | The exact shell command to run the test suite. `python` in this command gets replaced with the venv's Python automatically. Runs with `shell=True`, so quotes and pipes work. |
| `test_framework` | `str` | `"pytest"` | `pytest`, `unittest`, or `custom`. Informational. |
| `uses_xdist` | `bool` | `False` | Whether the project uses pytest-xdist for parallel testing. If true, the test command should include `-p no:xdist` to disable it -- parallel workers mask JIT-specific crashes. |
| `timeout` | `int \| None` | `None` | Per-package timeout in seconds. Null means use the runner's default (600s). Set higher for packages with large test suites. |

### Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `python_versions` | `list[str]` | `[]` | Python versions this package supports (informational). |
| `skip` | `bool` | `False` | Skip this package entirely during `labeille run`. |
| `skip_reason` | `str \| None` | `None` | Why the package is skipped -- build failures, missing 3.15 support, etc. |
| `skip_versions` | `dict[str, str]` | `{}` | Per-Python-version skip reasons. Keys are `"major.minor"` strings (e.g. `"3.15"`), values are human-readable reasons. When the target Python matches a key, the package is skipped -- unless `--force-run` is used. Use this instead of `skip: true` when a package only fails on specific Python versions. |
| `notes` | `str` | `""` | Free-form notes about quirks, known issues, or enrichment decisions. |
| `enriched` | `bool` | `False` | Whether this file has been reviewed and filled in. Set to `true` once you've determined the install and test commands, even if some tests fail. |

### Index fields

The index file (`index.yaml`) contains a summary entry for each package:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | (required) | Package name |
| `download_count` | `int \| None` | `None` | PyPI download count (used for sorting) |
| `extension_type` | `str` | `"unknown"` | Synced from the package YAML |
| `enriched` | `bool` | `False` | Synced from the package YAML |
| `skip` | `bool` | `False` | Synced from the package YAML |
| `skip_versions_keys` | `list[str]` | `[]` | Sorted list of Python version keys from `skip_versions` |

### Complete examples

A well-enriched package -- `click`, a pure Python CLI framework with a
straightforward test setup:

```yaml
package: click
repo: https://github.com/pallets/click/
pypi_url: https://pypi.org/project/click/
extension_type: pure
python_versions:
- '3.15'
- '3.14'
install_method: pip
install_command: pip install -e . && pip install pytest
test_command: python -m pytest tests/
test_framework: pytest
uses_xdist: false
timeout: null
skip: false
skip_reason: null
skip_versions: {}
notes: Click CLI framework by Pallets. Uses flit_core build backend.
enriched: true
clone_depth: null
import_name: null
```

A package with more complexity -- `setuptools-scm`, which needs git tags for
version detection and a pytest config override:

```yaml
package: setuptools-scm
repo: https://github.com/pypa/setuptools-scm/
pypi_url: https://pypi.org/project/setuptools-scm/
extension_type: pure
python_versions: []
install_method: pip
install_command: git fetch --tags --depth 1 && pip install -e . && pip install pytest setuptools tomli typing-extensions build
test_command: python -m pytest testing/ -o "strict_config=false"
test_framework: pytest
uses_xdist: false
timeout: 180
skip: false
skip_reason: null
skip_versions: {}
notes: SCM-based version management. Needs git tags for version detection
  (git fetch --tags). Tests in testing/ directory.
enriched: true
clone_depth: 50
import_name: setuptools_scm
```

## Enrichment Guide

### What is enrichment?

When you run `labeille resolve`, it creates skeleton YAML files for each
package -- a repo URL, an extension type classification, and a bunch of empty
fields. These skeletons are enough to know *what* to test, but not *how* to
test it. That's where enrichment comes in.

Enrichment is the process of filling in the installation and test instructions
for each package: what commands install it, what dependencies its test suite
needs, how to invoke the tests, and any quirks to work around. Without
enrichment, `labeille run` has nothing to run.

You can enrich packages manually, with Claude Code, or with another AI coding
agent. The process is the same either way -- read the project's configuration
files, figure out the right commands, write them into the YAML, and verify they
work.

**Enrichment is iterative.** You rarely get it right on the first try. Test
suites have undocumented dependencies, pytest configurations that conflict with
how labeille runs tests, and build systems that need special handling. The
workflow is:

1. Read the project's config files and fill in the YAML
2. Run the package through `labeille run`
3. Diagnose any failures from the output
4. Fix the YAML
5. Re-run until the test harness works

"Working" means the test suite runs -- exit code 0 (all tests pass) or exit
code 1 (some tests fail). Both are fine. What matters is that the harness is
correctly configured. Exit codes 2, 3, and 4 mean something is wrong with the
configuration.

### Enrichment workflow

```
1. Clone the repo
2. Run: labeille scan-deps /path/to/repo --package-name X --registry-dir registry/
3. Use the output to build install_command:
   - Start with: pip install -e . && pip install <resolved deps from scan>
   - Or if the project has test extras: pip install -e ".[test]" && pip install <missing deps>
4. Examine pyproject.toml / tox.ini ONLY for test configuration:
   - pytest addopts (plugins referenced? markers? flags?)
   - conftest.py fixtures (do they need special setup?)
   - xdist usage (set uses_xdist: true, add -p no:xdist)
5. Run labeille run --packages X to validate
6. Fix any remaining issues (typically config, not deps)
```

Before manually examining project files for test dependencies, always run
`labeille scan-deps` first:

```bash
labeille scan-deps /path/to/cloned/repo --package-name PACKAGE \
    --registry-dir registry/ --install-command "CURRENT_INSTALL_CMD"
```

This statically analyzes test imports and reports which pip packages are
needed. Use its output as the starting point for `install_command`, then
supplement with any deps needed for test configuration (pytest plugins
referenced in addopts, etc.).

If `scan-deps` reports unresolved imports:
- Check if they're local test modules (common: `dummyserver`, `testutils`,
  `helpers`, `fixtures`)
- If they're real packages with unusual import names, resolve them and
  consider adding the mapping to `src/labeille/import_map.py`

### Manual enrichment walkthrough

Let's walk through enriching a package from scratch using `packaging` as our
example -- it's pure Python, popular, and has a clean test setup.

#### Step 1: Examine the project

Start by looking at the project's configuration files to understand how it's
built and tested.

**Clone the repo and look at these files, in order:**

1. **`pyproject.toml`** -- The most important file. Check:
   - `[project.optional-dependencies]` for test/dev dependency groups
   - `[tool.pytest.ini_options]` for test configuration, addopts, and plugin flags
   - `[build-system]` to understand the build backend
   - `[tool.setuptools-scm]` or similar -- indicates version-from-git-tags

2. **`tox.ini` / `noxfile.py`** -- The test automation config. Shows which
   dependencies are installed for testing and how tests are invoked.

3. **`CONTRIBUTING.md` / `README.md`** -- Human-readable test instructions.
   Sometimes the only place unusual test requirements are documented.

4. **`requirements*.txt`** -- Some projects keep test dependencies in
   `requirements-test.txt` or `requirements-dev.txt`.

5. **`conftest.py` and a few test files** -- Check imports for dependencies
   that aren't listed anywhere else. This is the most common source of
   "mystery" `ModuleNotFoundError` failures.

For `packaging`, here's what we'd find:

- `pyproject.toml` has a `[project.optional-dependencies]` section with a
  `tests` group listing `pytest` and `pretend`
- The test directory is `tests/`
- No xdist, no special pytest plugins

#### Step 2: Determine the install command

Follow this decision tree:

1. **Is there a `[test]` or `[testing]` extra?**
   Use `pip install -e ".[test]"` -- but first check what it pulls in. If it
   includes heavy deps like numpy, rpds-py, or pydantic-core that won't build
   on 3.15, install deps manually instead.

2. **Is there a `[dev]` extra that includes test deps?**
   Use `pip install -e ".[dev]"` -- same caveat about heavy transitive deps.

3. **Are test deps only in `tox.ini` or `requirements-test.txt`?**
   Use `pip install -e . && pip install -r requirements-test.txt`

4. **None of the above?**
   Use `pip install -e . && pip install pytest <deps-from-test-imports>`

For `packaging`, option 1 works cleanly:

```
pip install -e ".[tests]"
```

But we could also go explicit (and this is what the actual YAML uses, to avoid
any transitive dependency surprises):

```
pip install -e . && pip install pytest pretend tomli_w
```

**Common pitfalls to watch for:**

- **Transitive dependency hell on 3.15.** Test extras that pull in rpds-py (via
  jsonschema >= 4.18), pydantic-core, or numpy will fail because these packages
  use PyO3/Rust and don't support 3.15 yet. Fix: install deps manually, use
  `jsonschema<4.18` (which uses pyrsistent instead of rpds-py), or skip deps
  you don't need.

- **setuptools-scm packages.** If the project uses setuptools-scm (check
  `[tool.setuptools-scm]` in pyproject.toml), shallow clones lose the git tags
  needed for version detection. Fix: set `clone_depth: 50` and add
  `git fetch --tags --depth 1` to the install command.

- **Editable installs that break.** Some packages (especially monorepo
  subdirectories) don't support `pip install -e .`. Fix: use
  `pip install .` (non-editable) instead.

- **Packages needing build deps upfront.** Some Cython or meson-based packages
  need build tools installed first:
  `pip install setuptools cython && pip install --no-build-isolation -e .`

#### Step 3: Determine the test command

Follow this decision tree:

1. **Does `pyproject.toml` have `[tool.pytest.ini_options]`?**
   It's a pytest project. Use `python -m pytest tests/` (or whatever test
   directory exists -- check for `tests/`, `test/`, `testing/`, or test files
   in the package directory itself).

2. **Is there a `tests/` directory with `test_*.py` files importing unittest?**
   Use `python -m unittest discover tests`

3. **Something else?**
   Check `tox.ini` or `noxfile.py` for the actual test invocation.

**Important checks for the test command:**

- **xdist usage.** If `addopts` includes `-n auto`, `--numprocesses=auto`, or
  the project lists pytest-xdist as a dependency, set `uses_xdist: true` and
  add `-p no:xdist` to the test command. Parallel workers mask JIT crashes.

- **Plugin flags in addopts.** If addopts includes `--cov`, `--timeout`, or
  other plugin flags, you have two choices:
  - Install the corresponding plugin (pytest-cov, pytest-timeout, etc.)
  - Override addopts entirely: `-o "addopts="`

- **Never use `-x` or `--exitfirst`.** A single unrelated test failure
  shouldn't prevent us from seeing JIT crashes in later tests.

- **Check `filterwarnings`.** If pytest config has `filterwarnings` entries
  that reference specific modules (e.g. `coverage.exceptions.CoverageWarning`),
  pytest tries to import those modules. They must be installed or the warning
  filter entries will error out.

For `packaging`, the test command is simple:

```
python -m pytest tests/
```

No xdist, no special plugins, no addopts conflicts.

#### Step 4: Test it locally

You can validate your enrichment manually:

```bash
# Clone and install
git clone --depth=1 https://github.com/pypa/packaging /tmp/test-packaging
cd /tmp/test-packaging
/path/to/jit-python -m venv /tmp/venv-packaging
source /tmp/venv-packaging/bin/activate
pip install -e . && pip install pytest pretend tomli_w

# Run tests with JIT enabled
PYTHON_JIT=1 PYTHONFAULTHANDLER=1 python -m pytest tests/
```

Or use labeille directly, which handles cloning, venv creation, and environment
setup for you:

```bash
labeille run --target-python /path/to/jit-python \
    --packages packaging \
    --repos-dir /tmp/repos --venvs-dir /tmp/venvs \
    -v
```

The `-v` flag shows verbose output including install logs and test output,
which is essential for diagnosing failures.

#### Step 5: Update the YAML and iterate

If the test run fails, diagnose the error from the output:

| Exit code | Meaning | Typical cause |
|-----------|---------|---------------|
| 0 | All tests passed | You're done! |
| 1 | Some tests failed | Config is correct -- tests just fail (expected on alpha Python) |
| 2 | Collection error | Missing test dependency (most common) |
| 3 | Internal pytest error | Strict config warnings, plugin conflicts |
| 4 | No tests collected | Wrong test path, unrecognized pytest arguments |
| 5 | No tests ran | Test discovery found nothing to run |
| 134 | SIGABRT | JIT crash -- this is what we're looking for! |
| 139 | SIGSEGV | JIT crash -- this is what we're looking for! |

For each failure type:

- **Exit 2 (ModuleNotFoundError):** Add the missing module to `install_command`
  and re-run with `--refresh-venvs`.
- **Exit 3 (internal error):** Usually a strict config issue. Try adding
  `-o "strict_config=false"` or `-o "addopts="` to the test command.
- **Exit 4 (no tests / bad args):** Check the test path. Also check if addopts
  includes flags that conflict with labeille's invocation.
- **Timeout:** Increase the `timeout` field, or test a subset of the test suite
  by specifying a subdirectory.

Set `enriched: true` once the test suite runs (exit 0 or 1). The package is
enriched even if some tests fail -- what matters is that the harness works
correctly.

If you changed `install_command`, use `--refresh-venvs` to force venv
recreation. If you only changed `test_command`, you don't need
`--refresh-venvs`.

### Common problems and solutions

A reference for frequent issues encountered during enrichment. These patterns
come from enriching ~350 packages.

#### Missing test dependencies (exit code 2)

The most common failure. Tests import modules that aren't listed in the install
command.

| Missing module | What it is | Packages that need it |
|----------------|------------|----------------------|
| `hypothesis` | Property-based testing | yarl, mpmath, rfc3339-validator, coverage |
| `trustme` | Local TLS certificates | requests, httpx, uvicorn |
| `objgraph` | Object graph debugging | greenlet, multidict |
| `psutil` | Process utilities | multidict, ipykernel |
| `testpath` | Path testing utilities | jeepney |
| `aioresponses` | Async HTTP mocking | google-auth (aio tests) |
| `pytest-cov` | Coverage plugin | many packages reference `--cov` in addopts |
| `pytest-timeout` | Timeout plugin | packages with `--timeout` in addopts |
| `setuptools` | Provides `pkg_resources` | legacy packages that import `pkg_resources` |
| `build` | PEP 517 build frontend | poetry-core integration tests |
| `tomli_w` | TOML writing | packaging, build |
| `covdefaults` | Coverage defaults | yarl (referenced in .coveragerc) |

**Fix:** Add the module to `install_command` and re-run with `--refresh-venvs`.

#### pytest addopts conflicts (exit code 3 or 4)

Many packages configure pytest addopts in `pyproject.toml` or `pytest.ini`
with flags that conflict with labeille's usage -- usually xdist flags or plugin
options.

| Package pattern | Problem | Fix |
|-----------------|---------|-----|
| `-n auto` or `--numprocesses=auto` in addopts | Requires pytest-xdist | Add `-p no:xdist -o "addopts="` |
| `--cov` in addopts | Requires pytest-cov | Install pytest-cov or `-o "addopts="` |
| `--timeout` in addopts | Requires pytest-timeout | Install pytest-timeout or `-o "addopts="` |
| `--strict-markers` in addopts | Errors on unknown markers | Keep it: `-o "addopts=--strict-markers"` |

The nuclear option is `-o "addopts="` which clears all configured addopts.
This fixes most pytest config conflicts but loses any intentional
configuration. Use it when you can't easily install all the required plugins.

#### Strict config errors (exit code 3)

Some packages enable `strict_config = true` in their pytest config, which
errors on any unrecognized configuration option.

**Fix:** Add `-o "strict_config=false"` to the test command.

#### Wrong rootdir detection

When a package repository has no pytest config file (`pyproject.toml`,
`pytest.ini`, etc.), pytest walks up the directory tree looking for one. If
labeille's own `pyproject.toml` is found first, pytest uses the wrong root
directory.

**Fix:** Add `--rootdir .` to the test command.

#### Wrong test paths (exit code 4)

The test directory might not be where you expect:

- `tests/` vs `test/` vs `testing/` (setuptools-scm uses `testing/`)
- Tests inside the package directory: `regex/tests/test_regex.py`
- Tests as a single file: `test_toml.py` in the project root

**Fix:** Check the actual directory structure and update the test command path.

#### Ignoring problematic test files

Sometimes specific test files or directories import unavailable dependencies
or test features we don't need. Rather than installing everything, ignore them:

```yaml
test_command: python -m pytest tests/ --ignore=tests/transport/aio --ignore=tests/integration
```

Common cases:
- `--ignore=tests/integration` -- integration tests that need running services
- `--ignore=tests/transport/aio` -- async transport tests needing aiohttp extras
- `--ignore=tests/supervisors` -- supervisor tests needing specific process setup

#### Build failures on Python 3.15

Some packages can't be built on 3.15 at all:

| Blocker | Affected packages | Status |
|---------|-------------------|--------|
| PyO3/maturin (Rust) | rpds-py, pydantic-core, orjson | Waiting for PyO3 3.15 support |
| Changed C API | Some Cython packages | Check case-by-case |
| Unreleased dependency pins | Packages pinning pre-release versions | Skip with reason |

**Fix:** For version-specific failures, use `skip_versions` instead of
`skip: true`:

```yaml
skip_versions:
  "3.15": "PyO3 not supported on 3.15"
```

This keeps the package testable on other Python versions. Use `skip: true`
only for packages that can never be tested (e.g. abandoned projects, broken
repos). Use `--force-run` to override both `skip` and `skip_versions` for
debugging.

#### Timeout issues

Some test suites are very large or have tests that hang:

| Package | Typical runtime | Notes |
|---------|-----------------|-------|
| setuptools | ~28 minutes | Very large test suite |
| networkx | ~14 minutes | |
| pyparsing | ~9 minutes | |
| typer | ~9 minutes | |
| lz4 | ~8 minutes | |

**Fix:** Set `timeout` to an appropriate value (in seconds). For extremely
large suites, consider testing a subset: `python -m pytest tests/unit/`
instead of the full suite.

#### YAML formatting tips

- **Keep commands on a single line.** Long lines are fine in YAML. Single-line
  commands are unambiguous and avoid multiline parsing confusion.
- **Verify YAML syntax after editing.** One broken YAML file crashes labeille
  for *all* packages (it loads all YAMLs at startup). Quick check:
  ```bash
  python -c "import yaml; yaml.safe_load(open('packages/mypackage.yaml'))"
  ```
- **Watch nested quotes.** Values with inner quotes like
  `-o "addopts=" -o "markers=custom: my marker"` need careful handling. Test
  the YAML loads correctly after editing.

### Working with Claude Code

Claude Code is excellent at the mechanical parts of enrichment -- reading
project configs, figuring out dependencies, iterating on failures. The human
should review the results and make judgment calls about skip decisions,
unusual setups, and whether a package is worth spending time on.

#### Initial enrichment prompt

For enriching packages that have skeleton YAML files (from `labeille resolve`):

```
Enrich the labeille registry files for the following packages:
{package list or "all unenriched packages in registry/packages/"}

For each package:
1. Clone the repo (use the URL in the YAML file).
2. Read pyproject.toml, tox.ini, CONTRIBUTING.md, and test files to determine
   the install command, test command, test framework, and any special needs.
3. Pay close attention to:
   - All optional-dependency groups (test, testing, dev, all)
   - pytest addopts that reference plugins (--cov, --timeout, etc.)
   - filterwarnings that reference modules that must be installed
   - setuptools-scm or other version-from-git-tags tools (set clone_depth: 50)
   - Whether the project uses xdist (set uses_xdist: true, add -p no:xdist)
   - Never include -x or --exitfirst in the test command
4. Update the YAML file and set enriched: true.
5. If the package can't be tested (needs external services, unreleased deps,
   PyO3 on 3.15), set skip: true with a clear skip_reason.

Refer to doc/enrichment.md for the full field reference and common pitfalls.
```

#### Iterative fix prompt

For fixing packages whose test runs failed:

```
Fix the labeille registry for these packages that failed their test runs:
{package list}

For each package, run:
  labeille run --target-python {path} --packages {name} \
      --repos-dir {repos-dir} --venvs-dir {venvs-dir} -v

Diagnose the failure from the verbose output:
- ModuleNotFoundError -> add the module to install_command
- pytest plugin/config error -> install the plugin or override addopts
- Build/install failure -> try different install approach
- Timeout -> increase the timeout field or test a subset

After fixing the YAML, re-run to verify. Use --refresh-venvs if you changed
the install_command (since the old venv has stale dependencies). Don't use
--refresh-venvs if you only changed the test_command.

Repeat until tests either run (pass or fail is fine -- we want the harness
working) or you determine the package can't be tested (set skip: true).

Refer to doc/enrichment.md for common problems and solutions.
```

#### Full enrichment-and-fix prompt

For doing everything in one go -- new packages, end to end:

```
Enrich and validate the labeille registry for these packages:
{package list}

Phase 1 -- Enrich: For each package, clone the repo and read the project
configuration to fill in the YAML fields. Set enriched: true. Refer to
doc/enrichment.md for field reference and enrichment rules.

Phase 2 -- Validate: Run each package through labeille:
  labeille run --target-python {path} --packages {name} \
      --repos-dir {repos-dir} --venvs-dir {venvs-dir} -v

If a package fails, diagnose the error, fix the YAML, and re-run. Use
--refresh-venvs when you change install_command. Iterate until the test
harness works or you determine it can't work (then set skip: true with
skip_reason).

Don't spend more than 3 attempts per package -- if it's still broken after
3 fixes, set skip: true with notes explaining what you tried.
```

### Tips for efficient enrichment

- **Start with pure Python packages.** Use `--skip-extensions` when running
  labeille. Pure Python packages are most likely to build on 3.15 and most
  useful for JIT testing (the JIT optimizes Python bytecode, not C extensions).

- **Batch similar packages.** Packages from the same organization (pallets,
  pypa, aio-libs, etc.) often have similar test setups, so you can reuse
  patterns.

- **Use persistent directories.** The `--repos-dir` and `--venvs-dir` options
  save significant time when iterating. You don't re-clone or re-create venvs
  on every run. Use `--refresh-venvs` only when you change install commands.

- **Set reasonable timeouts.** Use `--timeout` with 300-600 seconds to avoid
  getting stuck on packages with hanging tests.

- **Don't spend hours on one package.** When in doubt about a complex package,
  set `skip: true` with notes about what you tried, and move on. There are
  thousands of packages to test.

- **Scan test imports upfront.** Rather than using labeille as a dependency
  oracle (run, see ModuleNotFoundError, add dep, re-run), read the test files
  first and add all non-stdlib imports to the install command in one go. This
  saves hours of retry cycles across a large batch.

- **Run batches, not individual packages.** After enriching a batch, do a full
  run to catch any remaining issues:
  ```bash
  labeille run --target-python /path/to/jit-python \
      --packages pkg1,pkg2,pkg3 \
      --repos-dir /tmp/repos --venvs-dir /tmp/venvs \
      --timeout 600 -v
  ```

- **Use parallel execution for batch runs.** `--workers N` tests multiple
  packages in parallel, overlapping clone/install/test across packages:
  ```bash
  labeille run --target-python /path/to/python --workers 4
  ```

- **Check for orphaned processes after interrupting labeille.** Timeouts now
  kill the entire process group automatically. However, if you interrupt
  labeille itself (Ctrl+C), pytest subprocesses may survive and consume
  significant memory. Check with `ps aux | grep pytest` and clean up as needed.

- **Verify YAML after every edit.** One broken YAML file blocks all packages.
  A quick `python -c "import yaml; yaml.safe_load(open('file.yaml'))"` saves
  you from debugging a confusing "all packages failed" situation.

- **Update the registry index after editing package files:**
  ```bash
  labeille registry rebuild-index --registry-dir . --apply
  ```

## Enrichment Rules

The enrichment process distilled into three steps and ten rules, learned from
enriching the first 50 packages.

### Process

1. Clone repo, examine `pyproject.toml`/`setup.cfg`/`tox.ini`/`requirements*.txt`
   for test deps
2. Determine `install_command`, `test_command`, `test_framework`, `uses_xdist`,
   `timeout`
3. Write the package YAML, set `enriched: true`

### The 10 enrichment rules

1. **Cross-check test imports against installed deps** -- scan `conftest.py`
   and first test files for non-stdlib imports; common missed deps: `trustme`,
   `uvicorn`, `trio`, `tomli_w`, `appdirs`, `wcag_contrast_ratio`, `installer`,
   `setuptools`, `flask`
2. **Check pytest config for plugin flags** -- (`pyproject.toml`
   `[tool.pytest.ini_options]`, `tox.ini` `[pytest]`, `pytest.ini`) if addopts
   uses `--cov`/`--timeout`/etc, install the plugin (`pytest-cov`,
   `pytest-timeout`)
3. **Check filterwarnings for module references** -- (e.g.
   `coverage.exceptions.CoverageWarning`) pytest tries to import the module,
   so it must be installed
4. **For setuptools-scm packages, add `git fetch --tags --depth 1` to
   install_command** -- shallow clones lose tags and version detection fails;
   also set `clone_depth: 50`
5. **When `[test]` extras pull in heavy/problematic transitive deps** --
   (`numpy`, `rpds-py`, `pydantic-core`), install deps manually:
   `pip install -e . && pip install pytest <specific-deps>` instead of
   `pip install -e ".[test]"`
6. **For jsonschema dependency** -- (pulls `rpds-py` via PyO3): pre-install
   `jsonschema<4.18` which uses `pyrsistent` instead
7. **For packages whose main branch pins unreleased dependency versions** --
   set `skip: true` with `skip_reason`
8. **Never use `-x`/`--exitfirst`** -- a single unrelated failure shouldn't
   hide JIT crashes in later tests
9. **Always disable xdist for JIT testing** -- (`-p no:xdist`) parallel
   workers mask JIT-specific crashes
10. **For packages with `src/` layout** -- (e.g. `pytz`), verify install path
    -- may need `pip install src/` not `pip install -e src/`

### Common 3.15 alpha blockers

- **PyO3/maturin** (`rpds-py`, `pydantic-core`, `orjson`): won't build until
  PyO3 supports 3.15
- **meson-python** (`numpy`, `pandas`): need
  `pip install meson-python meson cython ninja` before `--no-build-isolation`
- **moto[server]**: requires `pydantic` -> `pydantic-core` (PyO3), blocks
  `aiobotocore` testing

## Registry Management

labeille provides CLI commands for batch operations on the registry. All
commands are under the `labeille registry` group.

### Commands

- `labeille registry sync` -- Clone or update the registry
- `labeille registry validate` -- Check YAML files against schema
- `labeille registry add-field` -- Add a field to all YAML files
- `labeille registry remove-field` -- Remove a field
- `labeille registry rename-field` -- Rename a field
- `labeille registry set-field` -- Set field values (with filters)
- `labeille registry rebuild-index` -- Rebuild `index.yaml`
- `labeille registry migrate` -- Run schema migrations

### Shared options

| Option | Description |
|--------|-------------|
| `--apply` | Actually write changes (without this, dry-run only) |
| `--lenient` | Skip files where the precondition isn't met instead of erroring |
| `--registry-dir PATH` | Path to registry directory (default: auto-detect) |
| `--where EXPR` | Filter packages (repeatable, combined with AND) |
| `--packages CSV` | Comma-separated package names |
| `--update-index / --no-update-index` | Rebuild the index after applying changes (default: true) |

### Examples

```bash
# Preview adding a new field
labeille registry add-field priority --type int --default 0 --after enriched

# Apply the change
labeille registry add-field priority --type int --default 0 --after enriched --apply

# Remove a field
labeille registry remove-field priority --apply

# Rename a field
labeille registry rename-field old_field new_field --apply

# Set timeout for specific packages
labeille registry set-field timeout 900 --packages setuptools,networkx --apply

# Set a field on all enriched packages
labeille registry set-field notes "" --where "enriched=true" --apply

# Validate all YAML files
labeille registry validate

# Validate with strict mode
labeille registry validate --strict
```

### Best practices

- Always dry-run first (omit `--apply`), review the preview, then re-run with
  `--apply`.
- Use `--lenient` when resuming an interrupted operation or when you expect
  some files to already have the field.
- Use `--after` to control field placement in the YAML for readability.
- Run `labeille registry validate` after batch edits to catch issues.

## Schema Migrations

Migrations are named transformations applied to registry YAML files. Each
migration is a Python function registered with `@register_migration` in
labeille's codebase.

### Key concepts

- **Dry-run by default**: Running a migration without `--apply` shows a
  preview of what would change, including sample results.
- **One-time execution**: Each migration runs once per registry. Re-application
  is blocked with the original date shown. To re-run, manually remove the
  entry from `migrations.log`.
- **Atomic writes**: Modified files are written atomically (write to temp file,
  then rename) to prevent corruption.
- **Log file**: Applied migrations are recorded as JSONL in `migrations.log`
  with timestamp and file counts.

### Usage

```bash
# List all migrations with status
labeille registry migrate --list

# Preview a migration (dry-run)
labeille registry migrate skip-to-skip-versions

# Apply a migration
labeille registry migrate skip-to-skip-versions --apply
```

### Built-in migrations

- **`skip-to-skip-versions`** -- Convert 3.15-specific `skip: true` entries to
  `skip_versions`. Detects version-specific skip reasons (PyO3, maturin, Rust,
  pydantic-core, rpds-py, Cython build failures, JIT crashes) using regex
  patterns.
- **`recover-no-tests-found`** -- Reset packages falsely skipped as "No test
  directory found" for re-enrichment.
- **`recover-no-repo-url`** -- Reset packages falsely skipped as "No source
  repository URL" for re-resolution.

### Adding new migrations

Create a function in labeille's `migrations.py` decorated with
`@register_migration`:

```python
@register_migration(
    "my-migration-name",
    "Human-readable description of what this migration does",
)
def my_migration(
    file_path: Path,
    data: dict[str, Any],
) -> MigrationResult:
    package = data.get("package", file_path.stem)

    if not some_condition(data):
        return MigrationResult(package=package, modified=False, description="not applicable")

    data["some_field"] = new_value

    return MigrationResult(
        package=package,
        modified=True,
        description="description of what changed",
    )
```

## Schema Versioning

The `schema.yaml` file in the registry root contains:

```yaml
schema_version: 1
min_labeille_version: "0.1.0"
```

labeille checks this at load time. If the registry's schema version is higher
than what labeille supports, it gives an actionable error asking the user to
update labeille. The schema version is incremented for backward-incompatible
schema changes. Registries without `schema.yaml` are accepted for backward
compatibility.

## Statistics

- 5,000+ packages tracked
- ~1,800 enriched with install/test commands
- ~720 active (not skipped, testable on current Python)
- Coverage of the top PyPI packages by download count
- 7 JIT crashes found so far: textual, fonttools, greenlet, ipython,
  markdown-it-py, pyasn1-modules, packaging

## Related Projects

- [labeille](https://github.com/devdanzin/labeille) -- The JIT bug hunter
  that consumes this registry
- [lafleur](https://github.com/devdanzin/lafleur) -- Evolutionary JIT fuzzer
  (complements labeille's approach)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for enrichment workflow, YAML
conventions, and how to submit changes.

## Acknowledgments

See [CREDITS.md](CREDITS.md).

## License

MIT -- see [LICENSE](LICENSE).
