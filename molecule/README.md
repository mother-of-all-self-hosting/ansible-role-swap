<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://ansible.readthedocs.io/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems:
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## What the suite can and cannot tell you

Read this before changing anything under `molecule/`. This role is unlike the others in this fleet: it manages swap, and swap is a property of the *kernel*, which a container shares with the machine running the tests.

Two of the three things the role does cannot be allowed to take effect inside a container:

- **`swapon` / `swapoff`.** If they succeeded, the machine running the suite would start swapping onto a file inside a container's overlay filesystem — and `molecule destroy` would then delete that file out from under it.
- **`vm.swappiness` and `vm.vfs_cache_pressure`.** These are global kernel parameters, not namespaced ones. Setting them from inside a container retunes the machine running the suite, permanently; nothing puts them back.

All scenarios therefore run **unprivileged** containers with **no systemd** (the role installs no units). Without `CAP_SYS_ADMIN` the kernel refuses `swapon` and `swapoff` outright, and Docker mounts `/proc/sys` read-only, so neither is possible. `prepare.yml` asserts both of those properties before it does anything else, so that an edit which made a platform privileged fails loudly instead of quietly swapping onto somebody's laptop. `prepare.yml` also refuses to run if the swap files of a scenario ever add up to more than 128 MB, because those are megabytes of zeroes written to a real disk on every scenario in the matrix.

That leaves the role unable to finish, so `prepare.yml` puts a recording shim in front of `swapon`, `swapoff` and `sysctl`, in `/usr/local/sbin` — which precedes `/usr/sbin`, where the real tools live, in root's `PATH`. The shims answer the way a real host would and record what they were asked for. Nothing takes their presence on trust: if a shim were not found, the real tool would run, fail (`EPERM`, or a read-only `/proc/sys`) and take `prepare.yml` or the converge down with it.

So what a green run proves is:

- everything the role writes to disk, for real: each swap file at its configured path, at its configured size, mode `0600` `root:root`, carrying the `SWAPSPACE2` signature only `mkswap` writes; its `/etc/fstab` entry, field by field, and that the system's own parser (`findmnt --verify`) still accepts the file afterwards; the content of `/etc/sysctl.d/99-swap.conf`;
- that the role asked `swapon`, `swapoff` and `sysctl` for exactly the right thing, exactly once per swap file — with the paths, the priorities and the parameter values it was configured with, not the ones it defaults to;
- that a second run changes nothing: each swap file comes through with the same inode, size, checksum and mtime, and nothing is given a second `swapon` or `swapoff`.

What it does not prove, and cannot without a virtual machine:

- **that the swap is actually usable.** The kernel never sees it. `verify.yml` asserts instead that the kernel's list of swap areas came through the run *unchanged* and never mentions the role's swap file. `/proc/swaps` belongs to the shared kernel, so it lists whatever the machine running the suite has enabled — a 4 GB `/mnt/swapfile` on a GitHub Actions runner, usually nothing on a laptop. That is the honest counterpart to the shim, and is also how a container that somehow gained the ability to touch that machine's swap would be caught.
- **that the kernel accepts the parameter values.** `vm.swappiness` is never really set; what is asserted is the value the role handed to `sysctl`, recovered from the shim's store.

The `default` and `uninstall` scenarios leave `idempotence` out of their test sequence. The role is idempotent where it matters — the whole initialization block is skipped once a swap file exists — but `ansible.posix.sysctl` reports a change whenever it reloads, so Molecule's own idempotence step would fail on something harmless. What a second run must not do is rewrite a swap file or repeat the destructive steps, and each of the two runs the role twice and asserts that directly instead. The `file-removal` scenario sets no kernel parameters, so it keeps Molecule's `idempotence` step and lets that be the second run.

## Scenarios

Currently these testing scenarios are available:

### `default`

`system_swap_enabled: true` on a host with no swap: everything `tasks/install.yml` does, for two swap files.

Every value the scenario sets differs from the role's own default — 48 MB rather than 1024, `/var/swap-molecule` rather than `/var/swap`, `vm.swappiness` at 33 rather than 10 — so that a value the role rendered from its variables can be told apart from one that merely looks right. `verify.yml` asserts the role's default path was *not* written to, and that each kernel parameter was applied as the scenario's value rather than the role's default or the deliberately wrong baseline `prepare.yml` seeded beforehand.

The first swap file is the one `system_swap_path`, `system_swap_size` and `system_swap_priority` describe, and the second is added through `system_swap_files_custom`, at a different size and with no priority. So the first has to reach `swapon` with `-p` and `/etc/fstab` with `pri=`, and the second has to be set up exactly the way the role set a swap file up before priorities existed. What `verify.yml` expects of each is spelled out by hand in `molecule.yml` rather than read back out of `system_swap_files`, so that the suite does not pass by looping over a list the role put together wrongly.

A decoy file is seeded next to the swap files, with the same mode, named nowhere in the role's variables, and is asserted to be untouched.

### `uninstall`

`system_swap_enabled: false` on a host that already has swap — the destructive half of the role, which a fresh-install scenario structurally cannot reach.

`prepare.yml` builds that host: three swap files, made the same way the role makes one and registered in `/etc/fstab` the same way, of which only two are named in `system_swap_files` — one through `system_swap_path`, the other through `system_swap_files_custom`. Afterwards exactly one file and exactly one `/etc/fstab` line must be left, and it must be the one the role was never told about. This is the scenario that pins down [the `state: present` / `state: absent` mix-up](https://github.com/mother-of-all-self-hosting/ansible-role-swap/commit/d948619) that used to leave a stale swap entry in `/etc/fstab` pointing at a file the role had just deleted.

It also records two things the role deliberately does not do: it leaves `/etc/sysctl.d/99-swap.conf` behind, so a host that turns swap off keeps whatever `system_swap_sysctl` last persisted; and it takes no second `swapoff` on a run where the swap files are already gone.

### `file-removal`

`system_swap_enabled: true` on a host that already has swap, with one entry in `system_swap_files` marked `state: absent` — the only way to take a single swap file away and keep the rest.

`prepare.yml` builds a host with three swap files. The role is told to keep the first, to remove the second and nothing about the third, and it is asked for a fourth that does not exist yet: what a host's variables look like while its swap is being moved from one file to another. Afterwards the first and the third must be untouched, the second gone and the fourth in place, in `/etc/fstab` as much as on disk.

The entry to remove is listed ahead of the one to create, and the two shims write to a single log, so that `verify.yml` can assert the order the role went about it in: `swapon` for the new file first, `swapoff` for the old one second. On a real host `swapoff` has to move whatever the file holds into memory and into the swap that is left. The new file is given a priority of 0, which is a priority like any other and the one a careless check for "is a priority set?" would drop.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name uninstall

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
