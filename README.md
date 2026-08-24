# [Ansible role wud](#wud)

Installs and configures WUD (What's Up Docker) container based on the official WUD docker image

|GitHub|Downloads|Version|
|------|---------|-------|
|[![github](https://github.com/mullholland/ansible-role-wud/actions/workflows/molecule.yml/badge.svg)](https://github.com/mullholland/ansible-role-wud/actions/workflows/molecule.yml)|[![downloads](https://img.shields.io/ansible/role/d/mullholland/wud)](https://galaxy.ansible.com/mullholland/wud)|[![Version](https://img.shields.io/github/release/mullholland/ansible-role-wud.svg)](https://github.com/mullholland/ansible-role-wud/releases/)|
## [Example Playbook](#example-playbook)

This example is taken from [`molecule/default/converge.yml`](https://github.com/mullholland/ansible-role-wud/blob/master/molecule/default/converge.yml) and is tested on each push, pull request and release.

```yaml
---
- name: Converge
  hosts: all
  gather_facts: true
  roles:
    - role: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}"
```

The machine needs to be prepared. In CI this is done using [`molecule/default/prepare.yml`](https://github.com/mullholland/ansible-role-wud/blob/master/molecule/default/prepare.yml):

```yaml
---
- name: Prepare
  hosts: all
  gather_facts: true
  vars:
    pip_packages:
      - "docker"

  roles:
    - role: mullholland.docker
    - role: mullholland.repository_epel
    - role: mullholland.pip
      when: not ((ansible_distribution == "Debian" and ansible_distribution_major_version | int >= 12) or
                 (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 24))

  tasks:
    - name: Install python3-docker on Debian 12+ / Ubuntu 24.04+
      ansible.builtin.apt:
        name: python3-docker
        state: present
      when: (ansible_distribution == "Debian" and ansible_distribution_major_version | int >= 12) or
            (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 24)

    # mullholland.pip only installs the pip binary itself, it no longer
    # installs pip packages (removed due to PEP 668), so the
    # community.docker modules used below need their Python dependency
    # installed here directly on every other distribution.
    - name: Install python dependencies for community.docker modules
      ansible.builtin.pip:
        name: "{{ pip_packages }}"
        state: present
      when: not ((ansible_distribution == "Debian" and ansible_distribution_major_version | int >= 12) or
                 (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 24))

    # Nested overlay2-on-overlay2 mounts can fail on some container backends
    # (e.g. OrbStack, some CI runners): "failed to mount ...: fstype: overlay
    # ... invalid argument". vfs avoids that at the cost of slower image
    # pulls, which is an acceptable trade-off for a molecule test instance.
    - name: Configure inner dockerd to use the vfs storage driver
      ansible.builtin.copy:
        dest: /etc/docker/daemon.json
        content: |
          {
            "storage-driver": "vfs"
          }
        mode: "0644"
      notify: restart docker

  handlers:
    - name: restart docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
```

See the [official WUD website](https://getwud.github.io/wud/) for more information about WUD itself (configuration reference, watchers, triggers).


## [Role Variables](#role-variables)

The default values for the variables are set in [`defaults/main.yml`](https://github.com/mullholland/ansible-role-wud/blob/master/defaults/main.yml):

```yaml
---
# General config
wud_docker_network_name: "web"
wud_docker_base_path: "/opt"
wud_docker_timezone: "Europe/Berlin"

# User/Group of the stack. Everything is mapped to this, instead of root.
wud_docker_user: "homelab"
wud_docker_uid: "900"
wud_docker_group: "homelab"
wud_docker_gid: "900"
wud_docker_user_system: true

# which container version to install
# can also be latest
wud_docker_version: "getwud/wud:latest"

# additional docker compose environment variables
# https://getwud.github.io/wud/#/configuration/
wud_docker_environment_variables:
  - "WUD_WATCHER_LOCAL_CRON: '0 * * * *'"       # check every hour for image updates
  # - "WUD_WATCHER_LOCAL_WATCHBYDEFAULT: false"  # only watch containers with wud.watch=true label, else watch all
  # WUD - Notifications (Triggers)
  # - "WUD_TRIGGER_DISCORD_MYDISCORD_TYPE: discord"
  # - "WUD_TRIGGER_DISCORD_MYDISCORD_URL: discord://token@channel"
  # web ui - basic auth
  # - "WUD_AUTH_BASIC_MYUSER_USER: admin"
  # - "WUD_AUTH_BASIC_MYUSER_HASH: $$apr1$$..."  # htpasswd hash

# which port to expose. disabled by default — exposing the web UI without auth is a security risk
# enable basic auth via WUD_AUTH_BASIC_* before uncommenting
wud_docker_ports: []
#  - "3000:3000"
wud_docker_labels:
  - "traefik.enable=false"
```

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/mullholland/ansible-role-wud/blob/master/requirements.txt).

## [State of used roles](#state-of-used-roles)

The following roles are used to prepare a system. You can prepare your system in another way.

| Requirement | GitHub | GitLab |
|-------------|--------|--------|
|[mullholland.repository_epel](https://galaxy.ansible.com/mullholland/repository_epel)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-repository_epel/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-repository_epel/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-repository_epel/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-repository_epel)|
|[mullholland.docker](https://galaxy.ansible.com/mullholland/docker)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-docker/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-docker/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-docker/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-docker)|
|[mullholland.pip](https://galaxy.ansible.com/mullholland/pip)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-pip/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-pip/actions)|[![Build Status GitLab](https://gitlab.com/mullholland-github-mirror/ansible-role-pip/badges/master/pipeline.svg)](https://gitlab.com/mullholland-github-mirror/ansible-role-pip)|

## [Context](#context)

This role is a part of many compatible roles. Have a look at [the documentation of these roles](https://mullholland.net) for further information.

## [Compatibility](#compatibility)

This role has been tested on these [container images](https://hub.docker.com/u/mullholland):

|container|tags|
|---------|----|
|[EL](https://hub.docker.com/r/mullholland/enterpriselinux)|all|
|[Fedora](https://hub.docker.com/r/mullholland/fedora/)|all|
|[Rocky](https://hub.docker.com/r/mullholland/rockylinux)|all|
|[AlmaLinux](https://hub.docker.com/r/mullholland/almalinux)|all|
|[Ubuntu](https://hub.docker.com/r/mullholland/ubuntu)|all|
|[Debian](https://hub.docker.com/r/mullholland/debian)|all|
|[CentOS](https://hub.docker.com/r/mullholland/centos)|all|

The minimum version of Ansible required is 2.10, tests have been done to:

- The version before the previous version.
- The previous version.
- The current version.

If you find issues, please register them in [GitHub](https://github.com/mullholland/ansible-role-wud/issues).

## [License](#license)

[MIT](https://github.com/mullholland/ansible-role-wud/blob/master/LICENSE).

## [Author Information](#author-information)

[mullholland](https://mullholland.net)
