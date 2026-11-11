<div align="center">
  <a href="https://libredevops.org">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://libredevops.org/assets/libre-devops-white.png">
      <img alt="Libre DevOps" src="https://libredevops.org/assets/libre-devops-black.png" width="300">
    </picture>
  </a>
</div>

# Ansible Role: Puppet agent

Runs, disables and enables an installed Puppet or OpenVox agent from Ansible. A run is `puppet agent --test`, with its detailed exit code turned into an Ansible result and the last run summary kept as a fact.

[![CI](https://github.com/libre-devops/ansible-role-puppet-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/libre-devops/ansible-role-puppet-agent/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/libre-devops/ansible-role-puppet-agent?sort=semver&label=release)](https://github.com/libre-devops/ansible-role-puppet-agent/releases/latest)
[![Galaxy](https://img.shields.io/badge/galaxy-libre__devops.puppet__agent-EE0000?logo=ansible&logoColor=white)](https://galaxy.ansible.com/ui/standalone/roles/libre_devops/puppet_agent/)
[![Licence](https://img.shields.io/github/license/libre-devops/ansible-role-puppet-agent)](./LICENSE)

---

## What it does

| `puppet_agent_action` | What happens |
|---|---|
| `run` (default) | Waits for any run already in progress, then runs `puppet agent --test` once in the foreground. `ok` when nothing changed, `changed` when Puppet changed resources, failed when the run failed or (by default) when resources failed. |
| `disable` | `puppet agent --disable "<message>"`. Changed only when the agent was enabled, or disabled with a different message. |
| `enable` | `puppet agent --enable`. Changed only when the agent was disabled. |

In check mode (`ansible-playbook --check`), or with `puppet_agent_noop: true`, a run is a noop run: Puppet changes nothing, and the task reports changed when resources are out of sync, so `--check` tells you what a real run would do.

It does not install, configure or upgrade the agent, or sign its certificate.

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Role variables](#role-variables)
- [How results are reported](#how-results-are-reported)
- [A disabled agent](#a-disabled-agent)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Developing](#developing)
- [Releasing](#releasing)
- [Sources](#sources)

## Requirements

| What | Detail |
|---|---|
| Control node | `ansible-core` 2.16 or later. No collections. |
| Managed nodes | A Puppet or OpenVox agent with a signed certificate, at `/opt/puppetlabs/bin/puppet` (the path both projects' packages use) or wherever `puppet_agent_bin` says. |
| Privileges | `become: true`, as for running the agent by hand. |

## Installation

From Galaxy:

```bash
ansible-galaxy role install libre_devops.puppet_agent
```

Or pin a release from GitHub, as in [`examples/requirements.yml`](./examples/requirements.yml).

## Quick start

```yaml
- name: Run the Puppet agent
  hosts: puppet_agents
  become: true
  serial: 10
  roles:
    - role: libre_devops.puppet_agent
```

```bash
ansible-playbook -i inventory.ini puppet.yml --check   # noop: what would change
ansible-playbook -i inventory.ini puppet.yml           # the real run
```

`serial` keeps the load on the Puppet server down and stops a bad change at the first batch.

## Role variables

Every variable is validated against [`meta/argument_specs.yml`](./meta/argument_specs.yml) when the role starts. Defaults are in [`defaults/main.yml`](./defaults/main.yml).

| Variable | Default | Description |
|---|---|---|
| `puppet_agent_action` | `run` | `run`, `disable` or `enable`. |
| `puppet_agent_bin` | `/opt/puppetlabs/bin/puppet` | The `puppet` command. |
| `puppet_agent_noop` | `false` | Run with `--noop`. Check mode does the same. |
| `puppet_agent_tags` | `[]` | Apply only resources with these tags (`--tags`). |
| `puppet_agent_environment` | `""` | Puppet environment for this run (`--environment`). Empty uses the agent's own. |
| `puppet_agent_server` | `""` | Puppet server for this run (`--server`). Empty uses the agent's own. |
| `puppet_agent_extra_args` | `[]` | More arguments for `puppet agent --test`, one per item, such as `--no-show_diff`. |
| `puppet_agent_when_disabled` | `fail` | What `run` does when the agent is disabled: `fail`, `skip` or `override`. See [A disabled agent](#a-disabled-agent). |
| `puppet_agent_wait_for_lock` | `600` | Seconds to wait for a run already in progress. `0` does not wait, and a run then fails with exit code 1 if one is in progress. |
| `puppet_agent_run_timeout` | `3600` | Seconds a run may take before Ansible stops waiting. |
| `puppet_agent_fail_on_resource_failures` | `true` | Fail the task when Puppet reports failed resources (exit codes 4 and 6). |
| `puppet_agent_disable_message` | `Disabled by Ansible` | The message for `disable`. The agent shows it at every run it skips. |

### Facts set by the role

After a `run`, `puppet_agent_result` holds:

| Key | Description |
|---|---|
| `rc` | The exit code of `puppet agent --test`. |
| `meaning` | What the exit code means, from the table below. |
| `noop` | Whether the run was a noop run. |
| `summary` | The agent's `last_run_summary.yaml`, parsed: `resources.changed`, `resources.failed`, `resources.out_of_sync`, `resources.total`, `time`, `version.config` and the rest. |
| `errors` | The `Error:` and `Warning:` lines from the run's output. |

## How results are reported

`--test` turns on `--detailed-exitcodes` ([OpenVox: puppet agent](https://docs.openvoxproject.org/openvox/latest/man/agent.html)), and the role maps each code:

| Exit code | Meaning | Ansible result |
|---|---|---|
| `0` | The run succeeded with no changes or failures. | ok |
| `1` | The run failed, or was not attempted because another run was in progress. | failed |
| `2` | The run succeeded, and some resources were changed. | changed |
| `4` | The run succeeded, and some resources failed. | failed, or ok with `puppet_agent_fail_on_resource_failures: false` |
| `6` | The run succeeded, and included both changes and failures. | failed, or changed with `puppet_agent_fail_on_resource_failures: false` |

A noop run exits `0` even when changes are pending, so for noop runs the role reads `resources.out_of_sync` from the last run summary instead, and reports changed when it is above zero.

`--test` is `--onetime --verbose --no-daemonize --no-usecacheonfailure --detailed-exitcodes --no-splay --show_diff`. Because of `show_diff`, the run's output (and so Ansible's) includes diffs of changed files. Pass `--no-show_diff` in `puppet_agent_extra_args` if that is a problem, and mark sensitive resources as `Sensitive` in your Puppet code either way.

## A disabled agent

`puppet agent --disable` writes a lock file with the reason, and the agent skips every run until someone enables it. The role reads that lock file before a run:

| `puppet_agent_when_disabled` | What `run` does |
|---|---|
| `fail` (default) | Fails, with the disable message, so nobody runs over someone else's work by accident. |
| `skip` | Leaves the agent alone and says why. |
| `override` | Enables the agent, runs it once, and disables it again with the same message, even when the run fails. |

## Examples

### Apply one change everywhere

```bash
ansible-playbook -i inventory.ini puppet.yml -e '{"puppet_agent_tags": ["ntp"]}' --check
ansible-playbook -i inventory.ini puppet.yml -e '{"puppet_agent_tags": ["ntp"]}'
```

### Try a branch environment on a few servers

```yaml
puppet_agent_environment: feature_new_ntp
puppet_agent_noop: true
```

### A change window

[`examples/maintenance.yml`](./examples/maintenance.yml) disables Puppet across a group with a message naming the change, leaves room for the work, then enables it and runs it once, five servers at a time.

### Use the result in later tasks

```yaml
- name: Run Puppet
  ansible.builtin.include_role:
    name: libre_devops.puppet_agent

- name: Restart the application if Puppet changed anything
  ansible.builtin.systemd_service:
    name: myapp
    state: restarted
  when: puppet_agent_result.summary.resources.changed | default(0) | int > 0
```

## Troubleshooting

| Symptom | What to check |
|---|---|
| `... does not exist on ...` | The agent is not installed, or is not at `/opt/puppetlabs/bin/puppet`. Set `puppet_agent_bin`. |
| `The Puppet agent on ... is disabled` | Someone ran `puppet agent --disable`. Read the message, then use `puppet_agent_when_disabled: override` or `skip`, or enable it with `puppet_agent_action: enable`. |
| Exit code 1 straight away | Another run was still going after `puppet_agent_wait_for_lock` seconds, the certificate is not signed, or the server is unreachable. `puppet_agent_result.errors` has Puppet's own messages. |
| Exit code 4 or 6 | Resources failed. `puppet_agent_result.errors` lists them, and `puppet agent --test` on the server shows the full output. |
| The task times out | The run took longer than `puppet_agent_run_timeout`. The run itself carries on on the server. |

## Developing

You need [uv](https://docs.astral.sh/uv/), [just](https://github.com/casey/just) and [podman](https://podman.io/). CI runs the same recipes.

| Recipe | What it does |
|---|---|
| `just deps` | Installs the pinned Python tools (`uv.lock`) and the collections in `requirements.yml` into `.ansible/`. |
| `just lint` | `yamllint`, `ansible-lint` on the production profile, and `zizmor` on the workflows. |
| `just test [scenario] [platform]` | Runs a Molecule scenario on one platform, or on all of them. `just test disabled ubuntu2404`. |
| `just test-all` | Every scenario on every platform, one after another. |
| `just converge <scenario> <platform>` | Converges and leaves the container running. `just login` opens a shell in it, `just destroy` removes it. |
| `just live <inventory> [args]` | Runs [`examples/playbook.yml`](./examples/playbook.yml) against real agents. |
| `just secrets` | Scans the git history with gitleaks. |
| `just platforms`, `just scenarios` | List what can be tested. |

There is no Puppet server in the tests, so [`molecule/fixtures/puppet`](./molecule/fixtures/puppet) stands in for the `puppet` command. It behaves as the agent documentation describes for the parts the role uses: `config print`, the disabled lock file and its JSON message, and the exit codes, with the first run changing resources and later runs changing none.

| Scenario | What it proves |
|---|---|
| `default` | `--check` gives a noop run, a run passes the tags and environment, the first run reports changed and the second ok. |
| `disabled` | A run on a disabled agent fails with its message, `override` runs it and puts the same message back, and disabling with the same message changes nothing. |

Test against real agents with `just live`.

## Releasing

1. Move the Unreleased entries in [`CHANGELOG.md`](./CHANGELOG.md) under the new version.
2. Run `just release 1.2.3`. It tags the commit and pushes the tag.
3. The Release workflow creates the GitHub release and imports the tag into Galaxy, which takes the version from the tag. It needs the `GALAXY_API_KEY` secret, which the libre-devops organisation provides. If the import fails, fix the cause and run the Release workflow by hand with the tag to import it again.

## Sources

| Topic | Source |
|---|---|
| `--test`, `--detailed-exitcodes`, `--disable`, `--enable` | [OpenVox: puppet agent man page](https://docs.openvoxproject.org/openvox/latest/man/agent.html), and [`lib/puppet/application/agent.rb`](https://github.com/OpenVoxProject/openvox/blob/main/lib/puppet/application/agent.rb) |
| The disabled lock file and its `disabled_message` | [`lib/puppet/agent/disabler.rb`](https://github.com/OpenVoxProject/openvox/blob/main/lib/puppet/agent/disabler.rb) |
| `agent_disabled_lockfile`, `agent_catalog_run_lockfile`, `lastrunfile` | [OpenVox: configuration reference](https://docs.openvoxproject.org/openvox/latest/configuration.html) |

## Licence

[MIT](./LICENSE).
