# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

## 1.0.0 - 2026-11-11

### Added

- First release.
- `run`: one `puppet agent --test` run, with tags, environment, server, extra arguments and noop (also from `--check`), reading the detailed exit code and the last run summary into the `puppet_agent_result` fact, and reporting changed when Puppet changed (or would change) resources.
- Waiting for a run already in progress, and a timeout for the run.
- `fail`, `skip` or `override` when the agent is disabled.
- `disable` with a message and `enable`, changed only when the state changes.
- Molecule scenarios on RHEL 9, Debian 13 and Ubuntu 24.04, with a stand-in `puppet` command.
