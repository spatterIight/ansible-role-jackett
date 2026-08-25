<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there is one testing scenario available.

### `default`

Installs Jackett and then checks that the installation is real rather than merely present.

The scenario deliberately configures the role with values nothing else would produce - uid/gid `1717`, the timezone `Asia/Tokyo`, the hostname `jackett.molecule.local`, the path prefix `/jackett-ui`, an extra read-only bind mount, an extra label and an extra container argument - and then looks for each of them on the running container.

It asserts, in order:

- the systemd unit is active. On its own this proves very little: the unit is `Restart=always`, so a container that crash-loops still reports `active`. It is a gate, not a result.
- Jackett answers its own `/health` endpoint
- the role's data path is bind-mounted at `/config`, and `jackett_container_additional_volumes` really became a `--mount`, read-only option included
- the container runs as `1717:1717` and carries the matching `PUID`/`PGID`
- `TZ` reached the container, **and** the process inside it reports the matching zone - the environment file alone would not prove that Jackett uses it
- the image is the tag `jackett_version` pins, and its `org.opencontainers.image.version` label belongs to that same version
- the Traefik labels carry the configured hostname, path prefix, both middlewares and port 9117; the additional label and the extra argument arrive too
- **Jackett's own log reports the version `defaults/main.yml` pins.** This is the assertion that lets Renovate automerge a patch bump: the bumped image has to be the one that ran.
- Jackett finished loading its indexer definitions, which happens after Kestrel starts listening
- the Torznab API serves a capabilities document to the API key Jackett wrote onto the mounted data path

The last check has a negative control next to it, because Jackett answers `200` to an unauthenticated Torznab call as well - it puts an `<error code="100" description="Invalid API Key" />` document in the body instead. The scenario makes the same call without a key and requires it to be rejected, so that the authenticated assertion cannot pass against an instance that refuses everything.

The API key is a real credential, so the tasks that read and use it are `no_log`.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
