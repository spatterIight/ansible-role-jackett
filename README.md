<!--
SPDX-FileCopyrightText: 2023 Slavi Pantaleev
SPDX-FileCopyrightText: 2025, 2026 Suguru Hirahara
SPDX-FileCopyrightText: 2025, 2026 spatterIight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Jackett Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [Jackett](https://github.com/Jackett/Jackett) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Limitations

This role configures Jackett with security in mind by doing the following:

1. Running the container as a non-root user
2. Making the filesystem read-only
3. Dropping most capabilities

Unfortunately, due to upstream requirements, some admissions had to be made:

1. Several capabilities related to permissions are added to the container
   - SETUID
   - SETGID
   - CHOWN
   - FOWNER
   - DAC_OVERRIDE
2. A `tmpfs` volume is mounted with `exec` permissions

You can read more about these upstream requirements in the documentation:

1. <https://docs.linuxserver.io/misc/non-root/>
2. <https://docs.linuxserver.io/misc/read-only/>

## Notes on configuration

- `jackett_container_http_port` describes the container image rather than configuring it. Jackett reads its listening port from the `ServerConfig.json` file it maintains on its own data path, and the container's readiness check is hardcoded to port 9117, so a container listening anywhere else would never come up. Changing the value only moves the Traefik label and the published port away from where Jackett actually listens.
- Jackett mints an API key on first start and keeps it, in plain text, in `ServerConfig.json` under the role's data path (`/jackett/data/Jackett/ServerConfig.json` by default). Jackett writes that file with mode `0644`; what keeps it private is the `0750` directory the role creates around it, owned by `jackett_uid`:`jackett_gid`. Anything you give that uid or gid to on the host can read the key, and the key is enough to drive the whole Jackett API.

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
