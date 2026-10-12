<!--
SPDX-FileCopyrightText: 2022 etke.cc
SPDX-FileCopyrightText: 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: GPL-3.0-or-later
-->

# Swap Ansible role

This is an [Ansible](https://www.ansible.com/) role which manages [swap](https://wiki.archlinux.org/title/Swap) files on a Linux host.

Unlike most other roles in this collection, this one does not run a container. It creates swap files on the host itself, enables them and registers them in `/etc/fstab`. It also sets the kernel parameters in `system_swap_sysctl`, which are `vm.swappiness` and `vm.vfs_cache_pressure` by default.

Swap files are assembled from 3 lists, in this order:

- `system_swap_files_default` — a single swap file, described by `system_swap_path`, `system_swap_size` and `system_swap_priority`. If one swap file is all you need, set those and nothing else.
- `system_swap_files_auto` — swap files contributed by a playbook or a group. You most likely do not want to set these yourself.
- `system_swap_files_custom` — additional swap files of your own. **This is the variable you want.**

Each swap file is a dictionary with `path` and `size` (in megabytes) keys, and optional `priority` and `state` keys:

- `priority` is a number from 0 to 32767. The kernel fills the swap file with the highest priority first and spreads pages evenly over those of the same priority. Without one, it uses the swap files one after another, in the order they were enabled.
- `state: absent` removes a swap file that was previously created: the role disables it, takes it out of `/etc/fstab` and deletes it. Dropping it from the list instead leaves it in place, as the role only acts on the swap files it is told about.

To move swap from one file to another in a single run, describe the new swap file and mark the old one as absent. The role enables the new one before it disables the old one:

```yaml
system_swap_path: /mnt/data/swap

system_swap_files_custom:
  - path: /var/swap
    state: absent
```

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Limitations

- **The size and the priority of an existing swap file are never changed.** The role only sets a swap file up when it is not there yet, on purpose: rewriting a swap file that the kernel is currently using would corrupt it. To change one, have the role remove it first, then set the new values and run it again.
- **`system_swap_enabled: false` removes every swap file in the lists**, whatever its `state`. It does not undo `system_swap_sysctl`: `/etc/sysctl.d/99-swap.conf` is left as the last run wrote it.
- **A swap file that could not be set up is left as it is.** When a step fails, the role stops working on that swap file, so that one which could not be enabled does not end up in `/etc/fstab`. With `system_swap_ignore_errors: true` it then carries on with the next swap file instead of failing the run. If the file had already been created by then, later runs skip it; delete it by hand to have it set up again.

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.

### Releases

Tags are created by [`.github/workflows/autotag.yml`](.github/workflows/autotag.yml), which asks [`bin/compute-next-tag.sh`](bin/compute-next-tag.sh) what the commit on `main` should be released as. This role deploys no software and has no version of its own, so the answer comes from the tags that already exist: any change under `defaults/`, `files/`, `meta/`, `tasks/` or `templates/` rolls the release counter (`v1.0.0-1`, `v1.0.0-2`, …), and a commit that only touches documentation, CI or the Molecule scenarios is not released at all. A new series is started by tagging one commit by hand, as `v2.0.0-0` say.

[`bin/test-compute-next-tag.sh`](bin/test-compute-next-tag.sh) exercises that script against throwaway repositories, and runs as a prek hook.
