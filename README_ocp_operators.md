# Mirror OpenShift Operator Images for a Disconnected Cluster

This repository provides playbooks that will mirror OpenShift operator images for a disconnected OpenShift installation.  The first playbook will mirror the operators to local disk and create a bundle file to transfer to the disconnected network.  On the disconnected network, a second playbook will be run to mirror the operators to the disconnected registry.

> NOTE: This process requires an accessible container registry when performing operator downloads on the `connected` host.  In a true disconnected environment, this requires two container registries, one accessible from `connected` and the actual disconnected host `registry`.

> This playbook was written specifically to work with Quay as the disconnected registry.  It may or may not work if the disconnected registry is something else.

## Infrastructure Prerequisites
1. Internet-connected host with podman installed
> Tested with:
>  * podman 2.2.1
>  * RHEL 8.3
2. Appropriate disk space for the operator images.  Depending on which and how many operators are mirrored, the disk space requirements can vary greatly.  You may need as little as 5G for one operator and upto 300G for multiple operators.
3. Mechanism to transfer bundle to disconnected network
4. Host on disconnected network running Quay
> Tested with:
> * Quay 3.4.3
## Connected Host Prerequisites
1. RHEL 8 host
2. openshift-client binary matching x.y `ocp_release`
2. opm binary matching x.y of `ocp_release`
## Disconnected Host Prerequisites
1. Quay credentials
2. Quay organization is already created
3. Quay credentials have write access to organization
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
> Pre-populated entries are set in `roles/olm-mirror/defaults/main.yml` and are ready to be used.  However, the values should be customized to your particular environment.  These variables are used for downloading and building the disconnected registsry tar file and for populating the disconnected registry itself.  The default values may be overridden any place that takes higher precedence.
1. `operator_list`: A list of dictionary values with `name` and `mirror` for each operator
> The intention was to allow the use of `mirror` to control whether a particular operator was acutally downlaoded for mirroring.  This may not be a viable solution, and you should set all operators to `mirror: yes` for the time-being.
2. `cleanup`: yes|no
> Using `cleanup: no` will preserve downloaded files so you can avoid having to wait for operators to downloaded when running this multiple times
3. `mirror_dir`: where operators will be downloaded to bundle for transfer
4. `bundle_dir`: where the bundled operator files are stored
> the filesystem for 3 and 4 should in total equal 2x the size of the operators being download (or equal size if different filesystems)
5. `operator_bundle`: full path of bundle file with all operators
6. `local_registry`: url (with port if necessary) of disconnected registry
7. `local_namespace`: must exist in registry
8. `pullsecret`: full path to pull-secret from https://cloud.redhat.com
9. `registry_secret`: full path to disconnected registry pull secret
10. `kubeconfig`: full path to kubeconfig file for disconnected cluster (needed to udpate `ImageContentSourcePolicy`)
> NOTE: there are other options for handling authentication to the cluster if desired
## Download Operator Images
Run the `download-operators.yml` playbook on the `connected` host.  This will create a tar file at `{{ bundle_file }}`.  Next, the tar.gz file should be transferred to `{{ bundle_file }}` on the `registry` host.  
### Prerequisites
> Ensure the approriate information is configured as defined in `roles/olm-mirror/defaults/main.yml` or the appropriate alternate location as required.
1. `disconnected_registry_user` has already been created with `disconnected_registry_pass` in the registry on `registry` (for Quay this should be a super-user)    
2. The username/password has been appropriately configured as a valid pull-secret and stored at `registry_secret`.
3. `local_namespace` has been created on `registry` when using Quay.  This should be an `organization`.  Ensure this is created with the same user as `disconnected_registry_user` or that `disconnected_registry_user` has the appropriate write permissions to the `organization`. 
> NOTE: For test environments, the `registry` host does not have to actually be disconnected and may actually be the same host used to do the initial mirror to disk.

### Execute Operator Download Playbook
```
# if using /etc/ansible/hosts
ansible-playbook download-operators.yml
```
```
# if using inventory.yml
ansible-playbook -i inventory.yml download-release.yml
```
```
# if using custom variable file
ansible-playbook download-operators.yml -e@myvars.yml
```
> NOTE: This ***must*** run on an internet connected host.  

This playbook will:
1. TBD
### Execute Upload Playbook
Upload by using mapping.txt directly, and to work around a bug in Quay, the mapping.txt is split into chunks of 10 lines to limit the rate data is uploaded.
```
# if using /etc/ansible/hosts or inventory configure in ansible.cfg
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
This playbook will:  
1. TBD