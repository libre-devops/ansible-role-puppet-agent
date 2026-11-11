# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

### Added

- `docs/using-without-galaxy.md`, linked from the README's Installation section: installing the role without Galaxy (git, release archives, requirements files that name their own source), getting its collections with no Galaxy access, keeping a copy on your own git server or artifact store, AWX, Automation Controller and execution environments, and the ansible-core and Python versions it needs.

### Changed

- The README's Requirements and Troubleshooting explain managed nodes with only Python 3.6 (RHEL 8, SLES 15), which ansible-core 2.17 and later cannot manage, and link to the two ways round it.
- The Release workflow imports the release's own tag into Galaxy, rather than the default branch, and can be run by hand with a tag to import that tag again.

## 1.0.0 - 2026-11-11

### Added

- First release.
- `run`: one `puppet agent --test` run, with tags, environment, server, extra arguments and noop (also from `--check`), reading the detailed exit code and the last run summary into the `puppet_agent_result` fact, and reporting changed when Puppet changed (or would change) resources.
- Waiting for a run already in progress, and a timeout for the run.
- `fail`, `skip` or `override` when the agent is disabled.
- `disable` with a message and `enable`, changed only when the state changes.
- Molecule scenarios on RHEL 9, Debian 13 and Ubuntu 24.04, with a stand-in `puppet` command.
