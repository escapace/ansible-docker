<p align="right">
    <a href="https://travis-ci.org/epiloque/ansible-docker">
        <img src="https://travis-ci.org/epiloque/ansible-docker.svg?branch=master"
             alt="build status">
    </a>
        <a href="https://galaxy.ansible.com/epiloque/docker">
        <img src="https://img.shields.io/badge/ansible--galaxy-docker-blue.svg"
             alt="ansible galaxy">
    </a>
</p>

docker role

## Role Variables

Available variables are listed below:

```yaml
# See https://github.com/spotify/docker-gc
docker_gc_install: true
# Docker log options
# See https://docs.docker.com/engine/admin/logging/overview/
docker_log_driver: journald
# Docker ipv6
# See https://docs.docker.com/engine/userguide/networking/default_network/ipv6/
docker_ipv6_enabled: false
docker_ipv6_cidr: "2001:db8:1::/64"
# Selinux
docker_selinux_enabled: false
# Storage options
# Possible values: overlay (default), devicemapper, btrfs (unsupported)
docker_storage_driver: overlay
# Refer to commentaries in ../templates/docker-storage-setup.conf.j2
# or `man lvcreate` for acceptable sizes, and their syntax
docker_volume_name: docker
docker_lvm_data_volume_size: 80%FREE
docker_lvm_data_volume_size_min: 2G
docker_lvm_auto_extend_pool: yes
```

## License

This software is released under the terms of the BSD-3-Clause license.

This software includes or is derivative of works distributed under the licenses
listed below. Please refer to the specific files and/or packages for more
detailed information about the authors, copyright notices, and licenses.

* [mantl](https://github.com/mantl/mantl) Copyright © 2015 Cisco Systems, Inc.
