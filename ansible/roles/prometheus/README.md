# Prometheus Role

Installs Prometheus as a binary from GitHub releases with sha256 checksum verification.

## Overview

This role downloads the Prometheus binary archive from the official [prometheus/prometheus](https://github.com/prometheus/prometheus) GitHub releases, verifies the sha256 checksum against the release's `sha256sums.txt`, and installs it as a systemd service.

By default it resolves the latest release version from the GitHub API. A specific version can be pinned by setting `prometheus_version` to a version string (e.g. `3.9.1`).

## Usage

Include the role in a playbook:

```yaml
- ansible.builtin.import_role:
    name: mttjohnson.infra.prometheus
  tags: prometheus
```

### Pinning a version

```yaml
# In inventory group_vars or host_vars
prometheus_version: "3.9.1"
```

### Enabling TLS with a self-signed certificate

```yaml
prometheus_tls_enabled: true
# prometheus_tls_mode defaults to "self_signed" — no further config needed
```

A 4096-bit RSA certificate valid for 10 years is generated on first run and stored in
`{{ prometheus_config_dir }}/tls/`. It is not rotated automatically; delete the files and
re-run the role to regenerate.

### Enabling TLS with an externally managed certificate (e.g. certbot)

```yaml
prometheus_tls_enabled: true
prometheus_tls_mode: symlink
prometheus_tls_cert_symlink_src: /etc/letsencrypt/live/prometheus.example.com/fullchain.pem
prometheus_tls_key_symlink_src: /etc/letsencrypt/live/prometheus.example.com/privkey.pem
prometheus_tls_insecure_skip_verify: false
```

The role creates symlinks at `{{ prometheus_tls_cert_dir }}/prometheus.crt` and
`prometheus.key` pointing to the source paths. The `prometheus` system user must be able to
read the private key at runtime. For certbot-managed certs the key is typically `0600
root:root`, so you will need a certbot deploy hook or group membership to grant read access
before the service starts.

### Enabling basic authentication

```yaml
prometheus_basic_auth_enabled: true
```

A random 32-character password is generated on the remote host on first run and stored at
`prometheus_basic_auth_password_file` (`/root/prometheus_basic_auth_password.txt`). The
bcrypt hash is cached at `prometheus_basic_auth_hash_file` so that `web-config.yml` stays
stable across playbook runs and does not trigger unnecessary restarts. The plaintext password
never leaves the remote host.

The self-scrape job is automatically configured to authenticate using a password file at
`prometheus_basic_auth_scrape_pass`.

> **Note:** Enabling basic auth without TLS sends credentials in plain text. Use both
> together for secure access.

### Default `latest` behavior

When `prometheus_version` is set to `latest` (the default), the role queries the GitHub API at `https://api.github.com/repos/prometheus/prometheus/releases/latest` to resolve the current release version. This request is unauthenticated and subject to GitHub's API rate limits. If you are running this frequently or in CI, consider pinning a version or setting a `GITHUB_TOKEN` environment variable on the controller.

## Variables

### Installation

| Variable | Default | Description |
|---|---|---|
| `prometheus_version` | `latest` | Version to install, or `latest` to auto-detect |
| `prometheus_binary_install_dir` | `/usr/local/bin` | Where to install `prometheus` and `promtool` binaries |
| `prometheus_config_dir` | `/etc/prometheus` | Configuration directory |
| `prometheus_db_dir` | `/var/lib/prometheus` | Time series data directory |
| `prometheus_system_user` | `prometheus` | System user to run the service |
| `prometheus_system_group` | `prometheus` | System group for the service user |
| `prometheus_system_user_extra_groups` | `[]` | Additional groups to add the system user to |

### Service

| Variable | Default | Description |
|---|---|---|
| `prometheus_web_listen_address` | `0.0.0.0:9090` | Listen address for the web UI and API |
| `prometheus_storage_retention` | `30d` | How long to retain time series data |
| `prometheus_storage_retention_size` | `0` | Maximum storage size (0 = unlimited) |
| `prometheus_extra_flags` | `[]` | Additional command-line flags passed to the prometheus binary |

### Self-scrape

| Variable | Default | Description |
|---|---|---|
| `prometheus_scrape_hostname` | `localhost` | Hostname used for the self-scrape target |
| `prometheus_scrape_port` | `9090` | Port used for the self-scrape target |

### TLS

| Variable | Default | Description |
|---|---|---|
| `prometheus_tls_enabled` | `false` | Enable HTTPS via Prometheus's native web config |
| `prometheus_tls_mode` | `self_signed` | Certificate source: `self_signed` or `symlink` |
| `prometheus_tls_insecure_skip_verify` | `true` | Skip TLS verification for the self-scrape job; set to `false` when using a CA-signed certificate |
| `prometheus_tls_cert_dir` | `{{ prometheus_config_dir }}/tls` | Directory holding the cert and key files |
| `prometheus_tls_cert_path` | `{{ prometheus_tls_cert_dir }}/prometheus.crt` | Path Prometheus reads the certificate from |
| `prometheus_tls_key_path` | `{{ prometheus_tls_cert_dir }}/prometheus.key` | Path Prometheus reads the private key from |
| `prometheus_tls_cert_symlink_src` | `""` | Source path for the certificate symlink (symlink mode only) |
| `prometheus_tls_key_symlink_src` | `""` | Source path for the private key symlink (symlink mode only) |

### Basic authentication

| Variable | Default | Description |
|---|---|---|
| `prometheus_basic_auth_enabled` | `false` | Require HTTP basic authentication for the web UI and API |
| `prometheus_basic_auth_username` | `prometheus` | Username for basic authentication |
| `prometheus_basic_auth_password_file` | `/root/prometheus_basic_auth_password.txt` | Remote path where the generated plaintext password is stored (root:root, 0600) |
| `prometheus_basic_auth_hash_file` | `/root/prometheus_basic_auth_password_hash.txt` | Remote path where the bcrypt hash is cached (root:root, 0600) |
| `prometheus_basic_auth_scrape_pass` | `{{ prometheus_config_dir }}/scrape-password` | Remote path where the plaintext password is stored for the self-scrape job |

## What gets installed

- **Binaries**: `prometheus` and `promtool` in `/usr/local/bin/`
- **Config**: `/etc/prometheus/prometheus.yml` (minimal self-scrape config)
- **Web config**: `/etc/prometheus/web-config.yml` (when TLS or basic auth is enabled)
- **TLS**: `/etc/prometheus/tls/prometheus.crt` and `prometheus.key` (when TLS is enabled)
- **Data**: `/var/lib/prometheus/`
- **Service**: `prometheus.service` systemd unit with security hardening (`ProtectSystem=strict`, `PrivateTmp`, `NoNewPrivileges`, etc.)
- **User**: `prometheus` system user with `nologin` shell

## Next steps

This role currently installs Prometheus with a minimal configuration that only scrapes itself. Areas to expand:

- **Scrape configuration** - Add scrape targets for node_exporter, application metrics, etc. The `prometheus.yml.j2` template can be extended with variables for `scrape_configs`, `rule_files`, and `alerting` sections.
- **Node Exporter** - Install [node_exporter](https://github.com/prometheus/node_exporter) on target hosts to collect system metrics.
- **Alertmanager** - Install [alertmanager](https://github.com/prometheus/alertmanager) and configure alerting rules.
- **Recording and alerting rules** - Add support for deploying rule files to `{{ prometheus_config_dir }}/rules/`.
- **File-based service discovery** - Add support for `file_sd_configs` target files for dynamic target management.
- **Reverse proxy** - Put Prometheus behind nginx/caddy with TLS termination using the existing certbot infrastructure.

## Reference

This role was written with reference to the [prometheus-community/ansible](https://github.com/prometheus-community/ansible) collection, specifically the [prometheus role](https://github.com/prometheus-community/ansible/tree/main/roles/prometheus). That collection provides a more comprehensive implementation with support for alerting rules, file service discovery, remote read/write, web config, and many other features. It is a useful reference for extending this role.
