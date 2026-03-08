# Contributing to laruche

Thanks for your interest! The most valuable contribution is enriching
packages -- filling in the install and test commands that make the
registry useful.

## Enriching Packages

### Quick version

1. Pick an unenriched package (search `index.yaml` for `enriched: false`)
2. Clone the package's source repo
3. Examine `pyproject.toml`, `tox.ini`, `setup.cfg` for test dependencies
4. Fill in `install_command` and `test_command` in the YAML file
5. Set `enriched: true`
6. Submit a PR

### Full guide

See the [Enrichment Guide](README.md#enrichment-guide) in the README
for the complete walkthrough, common problems, and tips.

## YAML Conventions

- Keep `install_command` and `test_command` on single lines (no YAML
  multiline syntax)
- Quote `notes` values that contain colons (YAML special character)
- Use `skip: true` with `skip_reason` for packages that can't be tested
- Use `skip_versions` instead of `skip: true` when only specific Python
  versions are affected
- Always include `-p no:xdist` in test commands to prevent parallel
  execution masking JIT crashes
- Never use `-x`/`--exitfirst` -- a single failure shouldn't hide crashes
  in later tests
- Valid `extension_type` values: `pure`, `c_extension`, `rust`,
  `cpp_extension` (not `extensions` or `unknown`)

## Reporting Issues

### Wrong or outdated package entries

If you find incorrect install/test commands, wrong repo URLs, or
outdated skip reasons:

1. Open an issue describing what's wrong
2. Or submit a PR with the fix

### New packages

To add packages not yet in the registry, use `labeille resolve`:

```bash
labeille resolve package-name
```

This creates a skeleton YAML file that you can then enrich.

## Development Setup

```bash
# Clone the registry
git clone https://github.com/devdanzin/laruche.git
cd laruche

# Install labeille (for validation and batch operations)
pipx install labeille
```

## Quality Checks

Before submitting a PR:

```bash
# Validate YAML files
labeille registry validate --registry-dir .

# Verify your YAML parses correctly
python -c "import yaml; yaml.safe_load(open('packages/mypackage.yaml'))"
```

## Pull Request Process

1. Fork the repository and create a branch
2. Make your changes (enrichments, fixes, or new packages)
3. Run `labeille registry validate --registry-dir .`
4. Submit a PR with a clear description of what you changed and why

## Commit Messages

We encourage [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: enrich click, flask, and requests packages
fix: correct test_command for setuptools-scm
docs: update README with new field descriptions
```

## AI-Assisted Enrichment

Contributors are welcome to use AI tools (Claude Code, Copilot, etc.) for
enrichment. The enrichment guide includes Claude Code prompt templates.
You are responsible for verifying that the generated commands actually work.

## Questions?

Open an issue or start a discussion. We're happy to help!
