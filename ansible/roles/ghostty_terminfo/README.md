# ghostty_terminfo

Ensure the `xterm-ghostty` terminfo entry is available system-wide on Debian/Ubuntu hosts so that SSH sessions from [Ghostty](https://ghostty.org/) — and follow-on `sudo -i`, `su -`, service account shells — render correctly instead of printing `WARNING: terminal is not fully functional`.

## What this role does

Ghostty sends `TERM=xterm-ghostty`. For ncurses-based tools to render correctly, the host needs a terminfo entry under that name. This role makes that true with the least-invasive path that works on the target:

1. **Always:** installs `ncurses-base`, `ncurses-bin`, and `ncurses-term` (essential for `infocmp` and the bundled terminfo database).
2. **If `xterm-ghostty` is already queryable** (e.g. a future ncurses-term ships it directly, or a prior role run installed it): no-op.
3. **If only the bare `ghostty` entry is present** (current upstream ncurses behavior — `ncurses-term` 6.5-20241228+ ships the entry under that name only): creates a symlink `/usr/share/terminfo/x/xterm-ghostty → ../g/ghostty`. No download.
4. **If neither name is present** (e.g. Ubuntu 24.04 with ncurses 6.4): downloads the Ghostty `.deb` from [github.com/mkasberg/ghostty-ubuntu](https://github.com/mkasberg/ghostty-ubuntu), extracts the precompiled `x/xterm-ghostty` file from inside it, installs it to `/usr/share/terminfo/x/xterm-ghostty`.

The role finishes with a verification step that re-runs `infocmp xterm-ghostty` and fails if the entry is still not queryable.

## Why a .deb and not the official Ghostty repo?

The upstream `ghostty-org/ghostty` repository does *not* ship a pre-rendered terminfo file. The entry is generated at build time from Zig source code (`src/terminfo/ghostty.zig`), and there is no release artifact you can download directly — see upstream issue [ghostty-org/ghostty#9917](https://github.com/ghostty-org/ghostty/issues/9917). The mkasberg/ghostty-ubuntu `.deb`, which the official Ghostty install docs point at, ships the compiled terminfo entry inside, so we extract that. (We do *not* install the .deb itself — only the ~4 KB terminfo file is copied out; the rest is discarded.)

The upstream **ncurses** project does ship a `ghostty` entry in `misc/terminfo.src` starting at 6.5-20241228 — but under the bare name `ghostty`, not `xterm-ghostty`. That's why mode `auto` does the symlink dance instead of using the ncurses entry directly.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `ghostty_terminfo_source` | `"auto"` | One of `auto`, `distro`, `extract_from_deb`. See below. |
| `ghostty_terminfo_deb_release` | `"1.3.1-0-ppa2"` | Release tag at `mkasberg/ghostty-ubuntu` used in `extract_from_deb`. |
| `ghostty_terminfo_deb_variant` | `"amd64_24.04"` | Which `.deb` variant to fetch. The terminfo file inside is identical across all variants of a given release; this only affects which URL is downloaded. |
| `ghostty_terminfo_deb_filename` | (computed) | Filename of the `.deb`. Override only for mirrors. |
| `ghostty_terminfo_deb_url` | (computed) | Full download URL. Override for internal mirrors. |
| `ghostty_terminfo_deb_checksum` | `sha256:707…91` of release `1.3.1-0-ppa2` | SHA256 of the `.deb`, `sha256:...` form. Update alongside `ghostty_terminfo_deb_release`. |
| `ghostty_terminfo_staging_dir` | `/var/cache/ansible-ghostty-terminfo/<release>` | Versioned staging path for the `.deb`. Cleaned up after extraction. |
| `ghostty_terminfo_compiled_path` | `/usr/share/terminfo/x/xterm-ghostty` | Final install path. |

### `ghostty_terminfo_source` modes

| Mode | Behavior | When to use |
|---|---|---|
| `auto` (default) | Prefer distro packaging; symlink when distro provides `ghostty` only; download `.deb` only when neither name is available. | Almost everyone. Becomes a true no-op as distros catch up. |
| `distro` | Same detection as `auto` but never falls back. Fails loudly if neither name is provided. | Air-gapped or policy-controlled hosts where the `.deb` download path is unacceptable. |
| `extract_from_deb` | Always download and extract the `.deb`, regardless of what the distro provides. Overwrites `x/xterm-ghostty`. | When you want the Ghostty-authored definition (newer or richer capability set than what ncurses ships) even on distros that already have an entry. |

## Network requirements

- `auto` mode on a current distro: only outbound apt traffic (no GitHub).
- `auto` mode on an older distro that falls through, or `extract_from_deb`: outbound HTTPS to `github.com` and `objects.githubusercontent.com` (or your mirror) to fetch the `.deb`.

The `.deb` is ~18 MB and is removed from `/var/cache/` after the ~4 KB terminfo file is extracted. The download only happens when this path is actually taken — `auto` on a distro that already provides the entry does no download at all.

## Supported platforms

Ubuntu LTS only: 22.04 (jammy), 24.04 (noble), 26.04 (resolute). Other Debian-family hosts and non-LTS Ubuntu interim releases likely work but are not explicitly targeted.

| Ubuntu LTS | ncurses version | Distro provides | What this role does in `auto` mode |
|---|---|---|---|
| 22.04 | 6.3 | neither | downloads `.deb`, extracts |
| 24.04 | 6.4 | neither | downloads `.deb`, extracts |
| 26.04 | 6.5+ | `g/ghostty` only | adds `x/xterm-ghostty → ../g/ghostty` symlink |

The `g/ghostty`-only behavior is consistent with current upstream ncurses; if a future LTS begins shipping `x/xterm-ghostty` directly, this role becomes a complete no-op there.

## Upgrading the pinned `.deb` version

1. Find the latest tag at https://github.com/mkasberg/ghostty-ubuntu/releases.
2. Compute the canonical-variant `.deb` SHA256:

   ```bash
   RELEASE=1.3.1-0-ppa2
   VERSION=${RELEASE%-*}.${RELEASE##*-}    # 1.3.1-0.ppa2
   curl -sSfL "https://github.com/mkasberg/ghostty-ubuntu/releases/download/${RELEASE}/ghostty_${VERSION}_amd64_24.04.deb" \
     | sha256sum
   ```

3. Update `ghostty_terminfo_deb_release` and `ghostty_terminfo_deb_checksum` together in your inventory / vars.

The compiled terminfo file is byte-identical across every arch/Ubuntu-version variant in a given release, so the `amd64_24.04` choice is purely a URL detail; the artifact installed on `arm64` hosts is identical.

## Example playbooks

Default behavior — best path automatically:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.ghostty_terminfo
```

Restrict to distro packaging only, fail rather than download:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.ghostty_terminfo
      vars:
        ghostty_terminfo_source: distro
```

Always use Ghostty's own definition from the `.deb`, even on distros that have an entry:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.ghostty_terminfo
      vars:
        ghostty_terminfo_source: extract_from_deb
```

Pin to a specific Ghostty release:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.ghostty_terminfo
      vars:
        ghostty_terminfo_source: extract_from_deb
        ghostty_terminfo_deb_release: "1.3.0-0-ppa1"
        ghostty_terminfo_deb_checksum: "sha256:<recomputed-for-1.3.0>"
```

## Out of scope

- Per-user terminfo (`~/.terminfo/`). System-wide install is sufficient for SSH, `sudo -i`, service accounts, etc.
- Installing the Ghostty client itself. This role only installs the terminfo description; no GUI app, no SSH config, no shell integration.
- Removal. There is no `state: absent`; if you want to uninstall, remove the file (and any symlink) by hand.
- Non-Debian families (RHEL, Arch, etc.).
- Offline operation with a bundled fallback `.terminfo` file. The role does not include a copy of the entry in `files/`; if you need offline operation, pre-stage the `.deb` under `ghostty_terminfo_staging_dir` or host it on an internal mirror and point `ghostty_terminfo_deb_url` at it.

## Related upstream tracking

- [ghostty-org/ghostty#9917](https://github.com/ghostty-org/ghostty/issues/9917) — request for Ghostty to publish the terminfo as a release artifact. Once that lands, this role could switch to pulling from a tiny official artifact instead of an 18 MB `.deb`.
- [ghostty-org/ghostty#2542](https://github.com/ghostty-org/ghostty/issues/2542) — tracking of upstream-ncurses inclusion. The entry is in ncurses now (under the bare `ghostty` name) and propagating into distros over time, so this role's `auto` mode increasingly converges on the symlink-only path and eventually a true no-op.

## License

MIT
