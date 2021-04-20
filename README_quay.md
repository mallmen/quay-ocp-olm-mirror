# Standalone Non-Production Quay for Disconnected OpenShift 4 Installation

This repository provides a playbook that will deploy a standalone instance of Quay using local storage on the container host.

> This repo is intended for Home Lab and Proof-of-Concept scenarios. All work for setting up Quay is based on the documentation located [here](https://access.redhat.com/documentation/en-us/red_hat_quay/3.4/html/deploy_red_hat_quay_for_proof-of-concept_non-production_purposes/index).

## Infrastructure Prerequisites

1. Host running podman to run Quay (this host can be connected or disconnected)
> Tested with:
>  * podman 2.2.1
>  * RHEL 8.3
2. Approximately 20G disk space available on host running Quay
> This should be sufficient space to mirror one release version of OpenShift 4 to Quay.  More space will be required to mirror additional versions and/or mirror operators.

## Disconnected Prerequisites

1. If Quay will be running on a disconnected host, the required images for Postgresql, Redis, and Quay must be pre-loaded into the local podman registry.

On an internet connected host:
```
podman login registry.redhat.io
podman pull registry.redhat.io/rhel8/postgresql-10:1
podman save registry.redhat.io/rhel8/postgresql-10:1 > /tmp/postgresql.tar 
podman pull registry.redhat.io/rhel7/redis-5:1
podman save registry.redhat.io/rhel7/redis-5:1 > /tmp/redis.tar
podman pull registry.redhat.io/quay/quay-rhel8:v3.4.3
podman save registry.redhat.io/quay/quay-rhel8:v3.4.3 > /tmp/quay.tar
```
Transfer the tar files to the disconnected host that will run Quay
```
podman load -i /path/to/postgresql.tar
podman load -i /path/to/redis.tar
podman load -i /path/to/quay.tar
```

2. Your pullsecret from [https://cloud.redhat.com](https://cloud.redhat.com) must exist on the container host as defined in `cloud_secret`.

## Setup
### Update Ansible inventory
In either `/etc/ansible/hosts` or a local `inventory.yml`, configure your inventory for your container host using a local connection.
```
registry:
  hosts:
    localhost:
  vars:
    ansible_connection: local
```
### Set Global Variables
* Quay registry variables
1. `base_dir`: A filesystem with at least 20G free for all Quay data to be stored.
> OPTIONAL: customize subdirectory names for Quay components and/or local pull secret
2. `registry_fqdn`: Container hostname (or an approriate alias)
3. `registry_ip`: IP of your container host
4. `#postgresql_ip`: Uncomment to use an IP different from `registry_ip`
> When testing on a container host running on VMware, quay could no communicate with postgresql without using the IP of the postgresql container.  In this case, set an IP address for postgresql to this variable to ensure the container always uses the same IP so the Quay config is always valid. (podman on RHEL 8 uses 10.88.0.x as the default container network)
5. `#redis_ip`: Uncomment to use an IP different from `registry_ip`
> When testing on a container host running on VMware, quay could no communicate with redis without using the IP of the redis container.  In this case, set an IP address for redis to this variable to ensure the container always uses the same IP so the Quay config is always valid. (podman on RHEL 8 uses 10.88.0.x as the default container network)
6. `quay_host_connected`: Set true|false if container host is internet-connected
> When set to ***true***, `required_pkgs` will be checked/installed and postgresql, redis, and quay containers will pull from registry.redhat.io.  When set to ***false***, postgresql, redis, and quay must be pre-loaded in the local podman registry.
7. `quay_port`: Set to desired SSL port
8. `cloud_secret`: full path for pullsecret from [https://cloud.redhat.com](https://cloud.redhat.com)


## Basic workflow
The `quay-setup.yml` playbook can be run at any time, and this will typically only be run once.
> NOTE: For test environments, the disconnected registry host does not have to actually be disconnected and may actually be the same host used to do the initial mirror to disk.

### Quay Registry Setup
`ansible-playbook quay-setup.yml` or `ansible-playbook -i inventory.yml quay-setup.yml`

1. Check for and install required packages as required
2. Setup and configure directories for Quay
3. Configure firewall if running
4. Create a self-signed SAN certificate
5. Create and start Postgresql container as required
6. Create and start Redis container as required
7. Create and start Quay container as required
