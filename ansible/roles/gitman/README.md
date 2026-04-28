# gitman

Install [gitman](https://gitman.readthedocs.io/) — a language-agnostic dependency manager that pulls and pins source-level dependencies from git repositories — at a pinned version via [pipx](https://pipx.pypa.io/).

https://gitman.readthedocs.io/

## Why this role exists

Infrastructure projects often need to pull in the source of another git repo (a shared Ansible role, a config bundle, a helper module) at a pinned commit, without going through a language-specific package manager. gitman handles that by reading a `gitman.yml` in the project, cloning the listed repos at the requested refs into a local directory, and verifying everything is at the expected commit.

Like Pipenv, gitman is a per-project tool whose binary is installed once per host. Pinning it via pipx and this role gives:

- **One gitman version per host** — controlled by `gitman_version`, promotable through dev → test → prod.
- **Isolation from system Python** — pipx puts gitman in its own venv.
- **No project state managed here** — `gitman.yml`, the `.gitman` cache, and `gitman install` invocations all live with the project repo.

## Install scope

- **Per-user**: gitman is installed via pipx into `~/.local/share/pipx/venvs/gitman` for `gitman_user`. Re-run the role per user to set up additional users.
- **No project files managed here**: this role does not create `gitman.yml`, run `gitman install`, or pre-populate the `.gitman` cache.
- **No shell integration here**: `gitman` lands in `~/.local/bin` (already on PATH after the `pipx` role runs).

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host
- `community.general` collection (for the `community.general.pipx` module)
- The [`pipx`](../pipx/) role having run first (declared as a dependency in `meta/main.yml`)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `gitman_version` | `"3.5.2"` | Exact gitman version installed via pipx. Bump and re-run to upgrade. Verify on PyPI before pinning. |
| `gitman_user` | `{{ ansible_facts['user_id'] }}` | User to install gitman for. Should match `pipx_user`. |
| `gitman_group` | `{{ ansible_facts['user_id'] }}` | Group ownership (currently unused — reserved). |

## Idempotency

The role queries `pipx list --short` for the user and only invokes `community.general.pipx` (with `force: true`) when the installed gitman version doesn't match `gitman_version`. Re-running with the same `gitman_version` is a no-op; bumping it triggers a clean reinstall at the new version.

## Example Playbooks

Default (install gitman for the connecting user at the pinned default):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.gitman
```

Pin a specific version for a dev user:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.gitman
      vars:
        gitman_version: "3.5.1"
        gitman_user: developer
        gitman_group: developer
```

Promote a gitman version through environments:

```yaml
# group_vars/dev.yml
gitman_version: "3.5.2"

# group_vars/prod.yml
gitman_version: "3.5.0"
```

## License

MIT
