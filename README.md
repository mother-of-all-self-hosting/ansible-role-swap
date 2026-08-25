<!--
SPDX-FileCopyrightText: 2022 etke.cc
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: GPL-3.0-or-later
-->

# system/swap

That role creates swap file and configures system to mount it and use automatically

## prerequisites

vars:

```yaml
system_swap_enabled: true
```

> **NOTE**: check [defaults/main.yml](./defaults/main.yml) to see full list of config options

## Testing

This role has a [Molecule](https://ansible.readthedocs.io/projects/molecule/) test suite. See [molecule/README.md](./molecule/README.md), which also explains what such a suite can and cannot prove about a role that manages kernel swap from inside a container.
