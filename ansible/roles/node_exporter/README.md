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
| `node_exporter_github_repo` | `prometheus/node_exporter` | GitHub repository used to resolve releases and download URLs |
| `node_exporter_binary_install_dir` | `/usr/local/bin` | Where to install the `node_exporter` binary |
| `node_exporter_config_dir` | `/etc/node_exporter` | Configuration directory |
| `node_exporter_system_user` | `node_exporter` | System user to run the service |
| `node_exporter_system_group` | `node_exporter` | System group for the service user |
| `node_exporter_system_user_extra_groups` | `[]` | Additional groups to add the system user to |
| `node_exporter_force_reinstall` | `false` | Set `true` to force re-download and reinstall even if the correct version is already installed |

### Service

| Variable | Default | Description |
|---|---|---|
| `node_exporter_web_listen_address` | `0.0.0.0:9100` | Listen address for the web UI and API |
| `node_exporter_extra_flags` | `[]` | Additional command-line flags passed to the node_exporter binary |
| `node_exporter_enabled_collectors` | `[]` | Non-default collectors to enable (e.g. `["systemd", "processes"]`) |
| `node_exporter_disabled_collectors` | `[]` | Default collectors to disable (e.g. `["mdadm"]`) |

### TLS

| Variable | Default | Description |
|---|---|---|
| `node_exporter_tls_enabled` | `false` | Enable HTTPS via node_exporter's native web config |
| `node_exporter_tls_mode` | `self_signed` | Certificate source: `self_signed` generates a self-signed cert; `symlink` creates symlinks to an externally-managed cert and key |
| `node_exporter_tls_cert_dir` | `{{ node_exporter_config_dir }}/tls` | Directory for certificate files |
| `node_exporter_tls_cert_path` | `{{ node_exporter_tls_cert_dir }}/node_exporter.crt` | Path to the TLS certificate |
| `node_exporter_tls_key_path` | `{{ node_exporter_tls_cert_dir }}/node_exporter.key` | Path to the TLS private key |
| `node_exporter_tls_cert_symlink_src` | `""` | Source path for cert symlink (used when `tls_mode == "symlink"`) |
| `node_exporter_tls_key_symlink_src` | `""` | Source path for key symlink (used when `tls_mode == "symlink"`) |

### Basic Auth

| Variable | Default | Description |
|---|---|---|
| `node_exporter_basic_auth_enabled` | `false` | Require HTTP basic authentication on the metrics endpoint |
| `node_exporter_basic_auth_username` | `node_exporter` | Username for basic auth |
| `node_exporter_basic_auth_password_hash` | `""` | Pre-computed bcrypt hash of the password |

> **Note:** Enabling basic auth without TLS sends credentials in plain text.

#### Generating a password hash

```bash
# Using htpasswd (apache2-utils)
htpasswd -nB username | cut -d: -f2

# Using Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'yourpassword', bcrypt.gensalt()).decode())"
```

### TLS + Basic Auth example

```yaml
node_exporter_tls_enabled: true
node_exporter_tls_mode: self_signed

node_exporter_basic_auth_enabled: true
node_exporter_basic_auth_username: node_exporter
node_exporter_basic_auth_password_hash: "$2y$12$..."  # bcrypt hash
```

### Collector configuration example

```yaml
# Enable non-default collectors
node_exporter_enabled_collectors:
  - systemd
  - processes

# Disable default collectors
node_exporter_disabled_collectors:
  - mdadm
```

### Check Mode Compatibility

This role is designed to work in check mode (`--check` flag). When the binary is not yet installed
or the version differs, a debug message reports what would be installed and all download/extract/copy
tasks are cleanly skipped (not errored) in check mode.

## What gets installed

- **Binary**: `node_exporter` in `/usr/local/bin/`
- **Config**: `/etc/node_exporter/` (directory always created; `web-config.yml` deployed when TLS or basic auth is enabled)
- **Service**: `node_exporter.service` systemd unit with security hardening (`ProtectSystem=strict`, `PrivateTmp`, `NoNewPrivileges`, etc.)
- **User**: `node_exporter` system user with `nologin` shell

