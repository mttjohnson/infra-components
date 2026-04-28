# pipx

Install [pipx](https://pipx.pypa.io/) — a tool for installing and running Python applications in isolated virtual environments — on Debian and Ubuntu hosts via the upstream `apt` package.

https://pipx.pypa.io/

## Why this role exists

pipx is the foundation other roles in this collection (e.g. [`pipenv`](../pipenv/), [`gitman`](../gitman/)) rely on to install Python CLIs into per-user, isolated venvs without touching system Python or polluting the user's environment with `pip install --user`. This role installs pipx itself and ensures the per-user `~/.local/bin` directory is on PATH so the apps installed by downstream roles are runnable.

## Install scope

- **System binary (bootstrap)**: `pipx` is first installed via `apt` to `/usr/bin/pipx`. On Ubuntu 24.04 (Noble) this is pipx 1.4.3 — fine as a bootstrap, but too old for `community.general.pipx` (which requires ≥ 1.7.0).
- **User-local pipx (the one downstream roles use)**: the role then uses `/usr/bin/pipx` to install a pinned newer pipx into `~/.local/bin/pipx` for the role's `pipx_user`. Downstream roles (`pipenv`, `gitman`) point `community.general.pipx`'s `executable` at this user-local binary.
- **Per-user state**: pipx stores managed venvs in `~/.local/share/pipx` and shims in `~/.local/bin`. This role configures one such user; re-run per user to set up additional users.

This role intentionally does **not** override `PIPX_HOME` or `PIPX_BIN_DIR` — pipx's defaults give the right per-user behavior on Ubuntu 24.04 and align with how `~/.local/bin` is already wired into the user's PATH via the stock `~/.profile`.

This role intentionally does **not** install any pipx-managed tools beyond the user-local pipx itself. That's the job of consumer roles (e.g. `pipenv`, `gitman`).

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host (designed against Ubuntu 24.04 / Noble; older Ubuntu releases ship older pipx versions but the role still works).

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `pipx_user` | `{{ ansible_facts['user_id'] }}` | Unprivileged user whose `~/.local/bin` is ensured on PATH and whose pipx state directory will be used by downstream roles. |
| `pipx_group` | `{{ ansible_facts['user_id'] }}` | Group ownership for files written into the user's home (currently unused — reserved for future per-user file management). |
| `pipx_user_version` | `"1.7.1"` | Version of pipx installed into the user's `~/.local/bin` via the apt-provided pipx. Must be ≥ 1.7.0 for `community.general.pipx` compatibility. |

## Idempotency

- `apt` install is idempotent.
- The user-local pipx bootstrap stats `~/.local/bin/pipx` and reads its `--version`; it only reinstalls (with `--force`) when the binary is missing or its version doesn't match `pipx_user_version`. Re-running with the same `pipx_user_version` is a no-op; bumping it triggers a clean reinstall.
- `pipx ensurepath` is a no-op on Ubuntu 24.04 because the stock `~/.profile` already prepends `~/.local/bin` to PATH when it exists; the task is registered with `changed_when: false` to reflect that. Running it explicitly keeps the role honest if the user's shell init is non-standard.

## Example Playbooks

Default (install pipx for the connecting user):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.pipx
```

For a specific dev user:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.pipx
      vars:
        pipx_user: developer
        pipx_group: developer
```

## License

MIT
