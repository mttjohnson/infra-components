# direnv

Install [direnv](https://direnv.net/) — a per-directory environment variable loader — on Debian and Ubuntu hosts via the upstream `apt` package, and wire up the per-user bash shell hook.

https://direnv.net/

## Why this role exists

direnv lets a project pin its own environment (PATH, env vars, virtualenv activation, secrets via `.envrc`) without polluting the user's global shell. It only takes effect after a shell hook is loaded into the user's rc file, so installing the binary alone does nothing useful — this role does both.

## Install scope

- **System binary**: `direnv` is installed via `apt` to `/usr/bin/direnv` and is available to all users.
- **Per-user shell hook**: an `# ANSIBLE MANAGED BLOCK - direnv` block is appended to the role's `direnv_user`'s shell rc file (default `~/.bashrc`). The block:
    - sources the bash hook via `eval "$(direnv hook bash)"`,
    - defines a `show_custom_ps1_prefix` helper, and
    - prepends `$(show_custom_ps1_prefix)` to `PS1` so per-project `.envrc` files can surface a tag in the prompt (either explicitly via `CUSTOM_DIRENV_PS1_PREFIX`, or implicitly when the project activates a Python venv that sets `VIRTUAL_ENV_PROMPT`).
- **Per-user direnv.toml**: when `direnv_quiet_logging: true` (default), `~/.config/direnv/direnv.toml` is written with `log_filter = "^$"` so direnv stops printing the INFO-level "loading .envrc" / variable-diff output on each `cd`. Errors from direnv still surface.

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host
- Bash as the target user's interactive shell (the injected block is bash-specific)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `direnv_user` | `{{ ansible_facts['user_id'] }}` | Unprivileged user to configure the shell hook for. |
| `direnv_group` | `{{ ansible_facts['user_id'] }}` | Group ownership for files written into the user's home. |
| `direnv_shell_rc` | `""` (→ `~/.bashrc`) | Shell rc file the init block is written into. |
| `direnv_quiet_logging` | `true` | Whether to write `~/.config/direnv/direnv.toml` to suppress direnv's INFO output. |
| `direnv_config_content` | `[global] log_filter = "^$"` (with commented `hide_env_diff` / `log_format` examples) | Verbatim content of `direnv.toml` when `direnv_quiet_logging` is true. Override to enable other knobs. |

## Idempotency

- `apt` install is idempotent.
- `blockinfile` matches its `# ANSIBLE MANAGED BLOCK - direnv` marker, so re-running with the same block is a no-op.
- `direnv.toml` is rewritten only when `direnv_config_content` differs from the file on disk, with `backup: true` so the previous version is preserved alongside it on change.

If the user has hand-written content in `direnv.toml` they want to keep, set `direnv_quiet_logging: false` and manage the file yourself — `copy` overwrites the whole file when this role manages it.

## PS1 customization

The injected block lets a per-project `.envrc` set a prompt prefix that only shows up while inside that directory tree:

```bash
# in a project's .envrc
export CUSTOM_DIRENV_PS1_PREFIX="(myproj) "
```

When `pipenv` (or any other tool) sets `VIRTUAL_ENV_PROMPT` via direnv, the block also surfaces that as the `(venv) ` marker without each `.envrc` having to set it explicitly.

## Example Playbooks

Default (install direnv, hook bash, quiet logging, for the connecting user):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.direnv
```

For a specific dev user:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.direnv
      vars:
        direnv_user: developer
        direnv_group: developer
```

Keep direnv's verbose load output (don't write `direnv.toml`):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.direnv
      vars:
        direnv_quiet_logging: false
```

Override the toml content (e.g. enable `hide_env_diff`):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.direnv
      vars:
        direnv_config_content: |
          [global]
          hide_env_diff = true
          log_filter = "^$"
```

## Using direnv after install

The shell hook takes effect on the user's next login (or `source ~/.bashrc`). Then in any project directory:

```bash
echo 'export FOO=bar' > .envrc
direnv allow .          # required once per .envrc per change
cd .                    # direnv loads FOO into the current shell
```

`direnv allow` is the security gate — direnv refuses to load an `.envrc` that has changed since you last approved it.

## License

MIT
