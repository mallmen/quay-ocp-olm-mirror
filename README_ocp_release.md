# Mirror OpenShift Release Images for a Disconnected Cluster

This repository provides playbooks that will mirror the OpenShift release images for a disconnected OpenShift installation.  The first playbook will mirror the images to local disk and create a bundle file to transfer to the disconnected network.  On the disconnected network, a second playbook will be run to mirror the images to the disconnected registry.

> This playbook was written specifically to work with Quay as the disconnected registry.  It may or may not work if the disconnected registry is something else.

## Infrastructure Prerequisites
1. Internet-connected host with podman installed
> Tested with:
>  * podman 2.2.1
>  * RHEL 8.3
2. Approximately 10G disk space available
> This should be sufficient space to mirror one release version of OpenShift 4 to Quay (as of 4.8.2).  More space will be required to mirror additional versions and/or mirror operators.
3. Mechanism to transfer bundle to disconnected network
4. Host on disconnected network running Quay
> Tested with:
> * Quay 3.4.3
## Disconnected Prerequisites
1. Quay credentials
2. Quay namespace and repository are already created
3. Quay credentials have write access to repository
4. Ensure certificate used by registry is trusted
## Setup
### Update Ansible inventory
In either `/etc/ansible/hosts` or a local `inventory.yml`, configure your inventory for your container host using a local connection.  Substitute for `localhost` as appropriate for the environment. `connected` should have internet connectivity.  
```
registry:
  hosts:
    localhost:
  vars:
    ansible_connection: local
connected:
  hosts:
    localhost:
  vars:
    ansible_connection: local    
```
### Set Global Variables
> Pre-populated entries are set in `roles/ocp-mirror/defaults/main.yml` and are ready to be used.  However, the values should be customized to your particular environment.  These variables are used for downloading and building the disconnected registsry tar file and for populating the disconnected registry itself.  The default values may be overridden any place that takes higher precedence.
1. `bin_dir`: Where to install `oc` and `openshift-install`, defaults to `/usr/bin`
2. `bundle_file`: This represents 2 locations
* Where bundle file is created locally on `connected` for transfer to `registry`
* Where bundle file is located on `registry` to be uploaded 
3. `removable_media_path`: 
* Where temporary images are downloaded on `connected` to create bundle file to transfer
* Where the bundle file is extracted on `registry` host for upload
4. `ocp_release`: x.y.z for OpenShift release to mirror
5. `local_repository`: namespace/repository in Quay to mirror *(**must already exist**)*
6. `cloud_secret`: full path for pullsecret from [https://cloud.redhat.com](https://cloud.redhat.com)
7. `registry_secret`: full path to created disconnected registry pull secret

### Username/Password Variables
> The credentials for the Quay registry should be stored with `ansible-vault`
1. `disconnected_registry_user`: your quay user
2. `disconnected_registry_pass`: your quay password
## Download Release Content
Run the `download-release.yml` playbook on the `connected` host.  This will create a tar file at `{{ bundle_file }}`.  Next, the tar.gz file should be transferred to `{{ bundle_file }}` on the `registry` host.  
### Prerequisites
> Ensure the approriate information is configured as defined in `roles/ocp-mirror/defaults/main.yml` or the appropriate alternate location as required.
1. `disconnected_registry_user` has already been created with `disconnected_registry_pass` in the registry on `registry` (for Quay this should be a super-user)    
2. `local_repository` has been created on `registry` when using Quay.  This should be an `organization` and a `repository`.  Ensure these are created with the same user as `disconnected_registry_user` or that `disconnected_registry_user` has the appropriate write permissions to the `organization` and `repository`. 
> NOTE: For test environments, the `registry` host does not have to actually be disconnected and may actually be the same host used to do the initial mirror to disk.

### Execute Download Playbook
Username/Password credentials for the Quay registry are not required at this stage.
```
# if using /etc/ansible/hosts
ansible-playbook download-release.yml
```
```
# if using inventory.yml
ansible-playbook -i inventory.yml download-release.yml
```
```
# if using custom variable file
ansible-playbook download-release.yml -e@myvars.yml
```
> NOTE: This ***must*** run on an internet connected host.  

This playbook will:
1. Setup directories for downloaded data
2. Download binaries for OpenShift release being mirrored
3. Install binaries on host system
4. Sync images to local directory
5. Create tar of images and binary downloads for transfer to disconnected registry host
6. OPTIONAL: Transfer to disconnected registry host if required

### Execute Upload Playbook
```
# if using /etc/ansible/hosts
ansible-playbook upload-release.yml
```
```
# if using inventory.yml
ansible-playbook -i inventory.yml upload-release.yml
```
```
# if using custom variable file
ansible-playbook upload-release.yml -e@myvars.yml
```
```
# if using custom variable file
ansible-playbook upload-release.yml -e@mycreds.yml --ask-vault-pass
```
This playbook will:  
1. Extract the transferred bundle
2. Install OpenShift binaries: oc, openshit-install
3. Create merged pull-secret including disconnected registry credentials
4. Sync OpenShift images from disk to registry
