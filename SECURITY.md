# Security policy

## Reporting a vulnerability

Report vulnerabilities in this role privately, through [GitHub's private vulnerability reporting](https://github.com/libre-devops/ansible-role-puppet-agent/security/advisories/new). Do not open a public issue.

## Things to know

- A run applies whatever the Puppet server's catalogue says, as root. The role adds no safety of its own beyond noop mode and the disable lock, so run noop first (`--check`) and use `serial` for large groups.
- `override` runs an agent that someone disabled on purpose. The role puts the same disable message back afterwards, but use it only when you know why the agent was disabled.
- Puppet's output can include diffs of managed files (`--test` turns on `show_diff`), and they end up in Ansible's output and logs. Mark sensitive resources `sensitive` in your Puppet code, or pass `--no-show_diff` through `puppet_agent_extra_args`.
