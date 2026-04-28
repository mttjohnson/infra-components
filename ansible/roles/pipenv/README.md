# pipenv

Install [Pipenv](https://pipenv.pypa.io/) at a pinned version via [pipx](https://pipx.pypa.io/), giving every project on the host a single shared Pipenv binary while keeping each project's own venv and `Pipfile.lock` independent.

https://pipenv.pypa.io/

## Why this role exists

Pipenv is a per-project tool: each repo has its own `Pipfile` / `Pipfile.lock` and its own venv. But Pipenv itself is a Python application, and you only want one copy of it on the host so multiple projects don't drift onto subtly different Pipenv versions. Installing it via pipx into the user's `~/.local/bin` gives:

- **One Pipenv version per host** — controlled by `pipenv_version` here, promotable through dev → test → prod.
- **Isolation from system Python** — pipx puts Pipenv in its own venv, so upgrading or removing it never touches anything else.
- **Lazy Python resolution** — when a project runs `pipenv sync`, Pipenv (which respects pyenv) picks the Python version pinned in the project's `Pipfile`. This role does **not** pre-install Python versions for you; the [`pyenv`](../pyenv/) role and the project's own `Pipfile` decide that.

## Install scope

- **Per-user**: Pipenv is installed via pipx into `~/.local/share/pipx/venvs/pipenv` for `pipenv_user`. Re-run the role per user to set up additional users.
- **No project state managed here**: this role does not create project directories, write `Pipfile` / `Pipfile.lock`, or run `pipenv sync`. Those live with the project repo.
- **No shell integration here**: `pipenv` lands in `~/.local/bin` (already on PATH after the `pipx` role runs). Per-project `pipenv shell` activation, `PIPENV_VENV_IN_PROJECT`, etc. are handled by `direnv` separately.

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host
- `community.general` collection (for the `community.general.pipx` module)
- The [`pipx`](../pipx/) role having run first (declared as a dependency in `meta/main.yml`)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `pipenv_version` | `"2026.6.0"` | Exact Pipenv version installed via pipx. Bump and re-run to upgrade. Verify on PyPI before pinning. |
| `pipenv_user` | `{{ ansible_facts['user_id'] }}` | User to install Pipenv for. Should match `pipx_user`. |
| `pipenv_group` | `{{ ansible_facts['user_id'] }}` | Group ownership (currently unused — reserved). |

## Idempotency

The role queries `pipx list --short` for the user and only invokes `community.general.pipx` (with `force: true`) when the installed Pipenv version doesn't match `pipenv_version`. Re-running with the same `pipenv_version` is a no-op; bumping it triggers a clean reinstall at the new version.

## Example Playbooks

Default (install Pipenv for the connecting user at the pinned default):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.pipenv
```

Pin a specific version for a dev user:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.pipenv
      vars:
        pipenv_version: "2024.4.0"
        pipenv_user: developer
        pipenv_group: developer
```

Promote a Pipenv version through environments:

```yaml
# group_vars/dev.yml
pipenv_version: "2024.4.1"

# group_vars/prod.yml
pipenv_version: "2024.3.1"
```

## License

MIT
