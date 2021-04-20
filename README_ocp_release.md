# Mirror OpenShift Release Images for a Disconnected Cluster

This repository provides playbooks that will mirror the OpenShift release images for a disconnected OpenShift installation.  The first playbook will mirror the images to local disk and create a bundle file to transfer to the disconnected network.  On the disconnected network, a second playbook will be run to mirror the images to the disconnected registry.

> This playbook was written specifically to work with Quay as the disconnected registry.  It may or may not work if the disconnected registry is something else.

## Infrastructure Prerequisites

1. Internet-connected host with podman installed
> Tested with:
>  * podman 2.2.1
>  * RHEL 8.3
2. Approximately 7G disk space available
> This should be sufficient space to mirror one release version of OpenShift 4 to Quay.  More space will be required to mirror additional versions and/or mirror operators.
3. Mechanism to transfer bundle to disconnected network
4. Host on disconnected network running Quay
> Tested with:
> * Quay 3.4.3

## Disconnected Prerequisites

1. Quay credentials 
2. Quay namespace and repository are already created
3. Quay credentials have write access to repository

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
* OpenShift mirror variables
> Pre-populated entries are set in **mirror-vars.yml** and are ready to be used.  However, the values should be customized to your particular environment.  These variables are used for downloading and building the disconnected registsry tar file and for populating the disconnected registry itself.
1. `bin_dir`: Where to install `oc` and `openshift-install`
2. `bundle_file`: Where bundle file is created to be transferred to disconnected registry host
                  Where bundle file is located on disconnected registry host
3. `removable_media_path`: Where temporary images are downloaded to create bundle file to transfer
                           Where the bundle file is extracted on disconnected registry host
4. `ocp_release`: x.y.z for OpenShift release to mirror
5. `local_repository`: namespace/repository in Quay to mirror *(**must already exist**)*
6. `cloud_secret`: full path for pullsecret from [https://cloud.redhat.com](https://cloud.redhat.com)
7. `merged_secret`: full path for merged pullsecret (cloud+disconnected_registry)
8. `disconnected_registry_user`: your quay user
9. `disconnected_registry_pass`: your quay password

## Basic workflow
Mirroring the OpenShift release images will be a two-stage operation.  First, run the `ocp-mirror-connected.yml` playbook.  This will create a tar file at `{{ bundle_file }}`.  Next, the tar.gz file should be transferred to `{{ bundle_file }}` on the disconneted registry host.  Ensure that the registry host has already been configured with a username/password and the namespace/reposistory with appropriate write permissions exists as defined in `mirror-vars.yml`.  Finally, the `ocp-mirror-disconnected.yml` playbook should be run on the disconnected registry.host.
> NOTE: For test environments, the disconnected registry host does not have to actually be disconnected and may actually be the same host used to do the initial mirror to disk.

### OpenShift image mirror disk
`ansible-playbook ocp-mirror-connected.yml` or `ansible-playbook -i inventory.yml ocp-mirror-connected.yml`
> This ***must*** run on an internet connected host.

1. Setup directories for downloaded data
2. Download binaries for OpenShift release being mirrored
3. Install binaries on host system
4. Sync images to local directory
5. Create tar of images and binary downloads for transfer to disconnected registry host
6. OPTIONAL: Transfer to disconnected registry host if required

### OpenShift image mirror to registry
`ansible-playbook ocp-mirror-disconnected.yml` or `ansible-playbook -i inventory.yml ocp-mirror-disconnected.yml`
1. Extract the transferred bundle
2. Install OpenShift binaries: oc, openshit-install
3. Create merged pull-secret including disconnected registry credentials
4. Sync OpenShift images from disk to registry

## TODO
Update `ocp-mirror-disconnected.yml` to include API calls to potentially create initial username/password and initial namespace/repoistory.
