<!--
SPDX-FileCopyrightText: 2022 etke.cc
SPDX-FileCopyrightText: 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: GPL-3.0-or-later
-->

# system/swap

That role creates swap files and configures system to mount them and use automatically

## prerequisites

vars:

```yaml
system_swap_enabled: true
```

> **NOTE**: check [defaults/main.yml](./defaults/main.yml) to see full list of config options

## Multiple swap files

The swap file described by `system_swap_path`, `system_swap_size` and `system_swap_priority` is the first one. More can be added with `system_swap_files_custom`:

```yaml
system_swap_files_custom:
  - path: /mnt/data/swap
    size: 4096
```

Each entry takes a `path`, a `size` in megabytes, and optionally:

- `priority`: a number from 0 to 32767, handed to `swapon` and written to `/etc/fstab` as `pri=`. The kernel fills the swap area with the highest priority first and spreads pages evenly over areas of the same priority. Without one, the kernel numbers the files itself and uses them one after another, in the order they were enabled.
- `state`: `present` (the default) or `absent`. An `absent` swap file is disabled, taken out of `/etc/fstab` and deleted.

**Deleting an entry from the list does not remove its swap file**: the role only acts on the files it is told about. Mark the entry `state: absent`, run the playbook, and delete the entry afterwards.

To move swap from one file to another in a single run, describe the new file and mark the old one `absent`. The role enables the new file before it disables the old one:

```yaml
system_swap_path: /mnt/data/swap

system_swap_files_custom:
  - path: /var/swap
    state: absent
```

## Notes

- **Changing the size or the priority of a swap file that already exists does nothing.** The role only sets a swap file up when it is not there yet, on purpose: rewriting a swap file that the kernel is currently using would corrupt it. To change one, have the role remove it first: set `system_swap_enabled: false` (which removes them all) or, for an entry in `system_swap_files_custom`, mark it `state: absent`, and run the playbook. Then set the new values and turn it back on.
- **`system_swap_enabled: false` removes every swap file in `system_swap_files`**, whatever its `state`.
- **A swap file that could not be set up is left as it is.** When a step fails, the role stops working on that swap file, so that one which could not be enabled does not end up in `/etc/fstab`. With `system_swap_ignore_errors: true` it then carries on with the next swap file instead of failing the run. If the file had already been created by then, later runs skip it; delete it by hand to have it set up again.
- **Turning swap off does not undo `system_swap_sysctl`.** `system_swap_enabled: false` removes the swap files and their `/etc/fstab` entries, but `/etc/sysctl.d/99-swap.conf` — and therefore `vm.swappiness` — is left as the last run set it.

## Testing

This role has a [Molecule](https://ansible.readthedocs.io/projects/molecule/) test suite. See [molecule/README.md](./molecule/README.md), which also explains what such a suite can and cannot prove about a role that manages kernel swap from inside a container.
