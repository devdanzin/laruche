# CLAUDE.md -- laruche development guide

## Project overview

laruche is a registry of installation and testing instructions for 5,000+
PyPI packages. It is consumed by
[labeille](https://github.com/devdanzin/labeille), which runs test suites
against JIT-enabled CPython builds to find compiler bugs.

## Repository structure

- `packages/{name}.yaml` -- Per-package YAML files with install/test commands
- `index.yaml` -- Summary index sorted by download count
- `migrations.log` -- Applied schema migration history
- `schema.yaml` -- Schema version for labeille compatibility

## Enriching packages

### Process

1. Clone the package repo, examine `pyproject.toml`/`setup.cfg`/`tox.ini` for
   test deps
2. Determine `install_command`, `test_command`, `test_framework`, `uses_xdist`,
   `timeout`
3. Write the package YAML, set `enriched: true`

### Enrichment rules

1. **Cross-check test imports against installed deps** -- scan `conftest.py` and
   first test files for non-stdlib imports; common missed deps: `trustme`,
   `uvicorn`, `trio`, `tomli_w`, `appdirs`, `wcag_contrast_ratio`, `installer`,
   `setuptools`, `flask`
2. **Check pytest config for plugin flags** -- (`pyproject.toml`
   `[tool.pytest.ini_options]`, `tox.ini` `[pytest]`, `pytest.ini`) if addopts
   uses `--cov`/`--timeout`/etc, install the plugin (`pytest-cov`,
   `pytest-timeout`)
3. **Check filterwarnings for module references** -- (e.g.
   `coverage.exceptions.CoverageWarning`) pytest tries to import the module, so
   it must be installed
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
9. **Always disable xdist for JIT testing** -- (`-p no:xdist`) parallel workers
   mask JIT-specific crashes
10. **For packages with `src/` layout** -- (e.g. `pytz`), verify install path --
    may need `pip install src/` not `pip install -e src/`

### Common 3.15 alpha blockers

- **PyO3/maturin** (`rpds-py`, `pydantic-core`, `orjson`): won't build until
  PyO3 supports 3.15
- **meson-python** (`numpy`, `pandas`): need
  `pip install meson-python meson cython ninja` before `--no-build-isolation`
- **moto[server]**: requires `pydantic` -> `pydantic-core` (PyO3), blocks
  `aiobotocore` testing

## YAML conventions

- Keep `install_command`/`test_command` on single lines
- Quote `notes` containing colons
- Valid `extension_type` values: `pure`, `c_extension`, `rust`, `cpp_extension`
- `skip: true` requires empty `''` commands; `skip: false` requires valid commands
- One broken YAML file crashes labeille for ALL packages -- always verify syntax
- `skip_versions` keys must be quoted strings: `"3.15"` (bare `3.15` parses as
  float)

## Registry batch operations

- `labeille registry sync` -- Clone or update the registry
- `labeille registry validate` -- Check YAML files against schema
- `labeille registry add-field` -- Add a field to all YAML files
- `labeille registry remove-field` -- Remove a field
- `labeille registry rename-field` -- Rename a field
- `labeille registry set-field` -- Set field values (with filters)
- `labeille registry rebuild-index` -- Rebuild `index.yaml`
- `labeille registry migrate` -- Run schema migrations

Best practices:
- Always dry-run first (omit `--apply`), review, then re-run with `--apply`
- Use `--lenient` when resuming interrupted operations
- Run `labeille registry validate` after batch edits

## Registry migrations

- `labeille registry migrate --list` shows available migrations
- `labeille registry migrate <name>` previews changes (dry-run default)
- `labeille registry migrate <name> --apply` applies and logs to
  `migrations.log`
- Each migration runs once -- re-application is blocked

## Useful commands

```bash
# Validate all YAML files
labeille registry validate --registry-dir .

# Rebuild index from package files
labeille registry rebuild-index --registry-dir . --apply

# Set a field on matching packages
labeille registry set-field skip_reason "needs CUDA" \
    --where "extension_type=c_extension" --registry-dir . --apply

# Find unenriched packages
grep "enriched: false" index.yaml | head -20
```
