# Pyenv

https://github.com/pyenv/pyenv
https://github.com/pyenv/pyenv-installer

This ansible role installs pyenv at the unprivileged user level so that the user can install and switch between multiple versions of python for development purposes.

## Usage

You may want to specify a few inventory host or group variables for how you want this installed. This doesn't create any users or groups it expects them to already exist and just install pyenv for that specific user. You can also have some versions of python installed during the initial setup if you want, and a default global version defined.

```yaml
pyenv_user: developer
pyenv_group: developer
pyenv_python_versions: ["3.14.4"]
pyenv_global_version: "3.14.4"
```
