# Node Exporter Role

Installs and configures the Node Exporter as a binary from GitHub releases with sha256 checksum verification.

## Overview

This role downloads the Node Exporter binary archive from the official [prometheus/node_exporter](https://github.com/prometheus/node_exporter) GitHub releases, verifies the sha256 checksum against the release's `sha256sums.txt`, and installs it as a systemd service.

By default it resolves the latest release version from the GitHub API. A specific version can be pinned by setting `node_exporter_version` to a version string (e.g. `1.8.0`).

## Usage

Include the role in a playbook:

```yaml
- ansible.builtin.import_role:
    name: mttjohnson.infra.node_exporter
  tags: node_exporter
```

### Pinning a version

```yaml
# In inventory group_vars or host_vars
node_exporter_version: "1.8.0"
```

## Variables

### Installation

| Variable | Default | Description |
|---|---|---|
| `node_exporter_version` | `latest` | Version to install, or `latest` to auto-detect |
| `node_exporter_binary_install_dir` | `/usr/local/bin` | Where to install the `node_exporter` binary |
| `node_exporter_config_dir` | `/etc/node_exporter` | Configuration directory |
| `node_exporter_system_user` | `node_exporter` | System user to run the service |
| `node_exporter_system_group` | `node_exporter` | System group for the service user |
| `node_exporter_system_user_extra_groups` | `[]` | Additional groups to add the system user to |

### Service

| Variable | Default | Description |
|---|---|---|
| `node_exporter_web_listen_address` | `0.0.0.0:9100` | Listen address for the web UI and API |
| `node_exporter_extra_flags` | `[]` | Additional command-line flags passed to the node_exporter binary |

### Checks Mode Compatibility

This role is designed to work in check mode (`--check` flag) to allow for review of changes before applying them. All tasks support check mode with proper `ignore_errors: "{{ ansible_check_mode }}"` configuration.

## What gets installed

- **Binary**: `node_exporter` in `/usr/local/bin/`
- **Config**: `/etc/node_exporter/` (empty by default, but directory created)
- **Service**: `node_exporter.service` systemd unit with security hardening (`ProtectSystem=strict`, `PrivateTmp`, `NoNewPrivileges`, etc.)
- **User**: `node_exporter` system user with `nologin` shell

## Next steps

- Consider adding support for systemd socket activation
- Add configuration file support for enabling specific collectors
- Add support for managing node_exporter configuration via templates