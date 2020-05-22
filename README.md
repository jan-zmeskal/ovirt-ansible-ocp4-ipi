ovirt-ansible-ocp4-ipi
=========

This role automates deployment of OpenShift 4 IPI (installer-provided infrastructure) on top of oVirt.
As of April 2020, oVirt is in developer preview as an OpenShift provider.
The IPI installation using `openshift-install` is admittedly quite easy
in itself and requires only minimal manual intervention.
Therefore this role is mainly intended for those who need to perform it repeatedly,
possibly on many oVirt engines (quality engineering, performance testing etc.)

Requirements
------------

- oVirt >= 4.3.9

This role is intended to be executed against your oVirt engine.
You need to execute it as the root user, for example by specifying `ansible_user: root` on host level
or `remote_user: root` on play level.
As long as you use oVirt engine as the target hosts, all of the below requirements should be in place:

- Python SDK version >= 4.3
- Ansible >= 2.9

You also need to authenticate with oVirt engine before executing this role.
Use ansible module [ovirt_auth](https://docs.ansible.com/ansible/latest/modules/ovirt_auth_module.html) to do that.

Role variables
--------------

> :open_book: Read the official documentation before trying to determine values of the following vairables.

| Name | Default value | Description |
|------|---------------|-------------|
| ocp_base_domain | UNDEF | DNS domain of your OCP cluster. |
| ocp_cluster_name | UNDEF | The name of your OCP cluster. This name will be included in the resulting domain of apps running on OCP like this: `console-openshift-console.apps.<ocp_cluster_name>.<ocp_base_domain>`. |
| api_vip | UNDEF | Virtual IP of OpenShift API. You should have `api.<ocp_cluster_name>.<ocp_base_domain>.` entry for this IP in your DNS setup. |
| dns_vip | UNDEF | Virtual IP of internal DNS for OpenShift cluster. |
| ingress_vip | UNDEF | Virtual IP of OpenShift Ingress. You should have `*.apps.<ocp_cluster_name>.<ocp_base_domain>.` entry for this IP in your DNS setup. |
| ovirt_cluster_name | UNDEF | The name of oVirt cluster that will host your OpenShift cluster |
| ovirt_storage_domain_name | UNDEF | The name of storage domain that will host your OpenShift cluster. Pay special attention to storage requirements of `etcd`. Read more [here](https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd) |
| ovirt_network_name | ovirtmgmt | The oVirt network of deployed OpenShift nodes (VMs). |
| ovirt_vnic_profile_name | ovirtmgmt | The oVirt vNIC profile you want to use with your network. |
| engine_url | UNDEF | URL of your engine's API in format `https://<engine_fqdn>/ovirt-engine/api`. |
| engine_user | UNDEF | The oVirt user that will be used to provision oVirt infrastructure for OpenShift cluster. |
| engine_password | UNDEF | Password of the abovementioned oVirt user. |
| custom_rhcos_template | UNDEF | If you provide a template name to this variable, specified template will be used for OpenShift installation instead of the default one. If you specify this, you also need to specify `rhcos_node_memory_mb` and `rhcos_node_cpu`. |
| rhcos_node_memory_mb | UNDEF | The amount of memory in MB that OpenShift nodes should have when using custom template. |
| rhcos_node_cpu | UNDEF | The count of logical CPUs that OpenShift nodes should have when using custom template. |
| master_count | 3 | Number of master machines (control plane) in OpenShift cluster. Keep in mind that having less than three masters is not supported. |
| worker_count | 3 | Number of worker nodes in OpenShift cluster. |
| pull_secret_file | UNDEF | The file containing your pull secret that can be obtained at [cloud.redhat.com](https://cloud.redhat.com/). |
| ssh_keys | UNDEF | List of strings. Each of the strings is a public SSH key that will be propagated to OpenShift RHCOS nodes, allowing the user to log in. |
| ocp_installer_source | nightly | Where to acquire `openshift-install` tool. At this point, only nightly is supported. |
| oc_source | nightly | Where to acquire `oc` tool. At this point, only nightly is supported. |
| ocp_installer_version | 4.4 | What version of OpenShift you want to install. oVirt as OpenShift provider is in developer preview now (April 2020). |
| ocp_installer_log_level | debug | Verbosity of OpenShift installer. This is passed to `--log-level` parameter of `openshift-install`. |

Customizing OpenShift nodes
---------------------------

Since this [PR](https://github.com/openshift/installer/pull/3399) OpenShift installer lets you customize your OpenShift on RHV nodes (i.e. RHV VMs).
More info on how can you do that while manually using the installer is available [here](https://github.com/openshift/installer/blob/master/docs/user/ovirt/customization.md).
To use that feature in this role, following variables are available:
  - custom_master_cpu_sockets
  - custom_master_cpu_cores
  - custom_master_memory_mb
  - custom_master_instance_type_id
  - custom_master_disk_size_gb
  - custom_master_vm_type
  - custom_worker_cpu_sockets
  - custom_worker_cpu_cores
  - custom_worker_memory_mb
  - custom_worker_instance_type_id
  - custom_worker_disk_size_gb
  - custom_worker_vm_type

Additional tasks
----------------

The role also performs some quality-of-life related tasks. Overwrite the following variables to determine if they
should be executed or not.

| Name | Default value | Description |
|------|---------------|-------------|
| perform_dns_check | true | # If set to true, role will fail early if you haven't set up DNS entries for API and Ingress |
| include_engine_ssh_key | true  | If set to true, public SSH key of engine machine will be included in RHCOS nodes. It will be also generated, if it doesn't exist. |

Flow control
------------

The role has few variables that control its flow. They have been implemented during testing and you probably won't feel any need to touch those.

| Name | Default value | Description |
|------|---------------|-------------|
| stop_before_installation | false | If set to true, the installation won't be run, but everything will be prepared. |
| download_openshift_install | true | This is required unless `stop_before_installation` is set to true. |
| download_oc_tool | false | If set to true, downloads the oc tool and puts it into installation folder. |

Example Playbook
----------------

You might notice that some of the required variables are missing here. That's because I specify them on per-host
basis in my inventory file. Specifically, I set following variables on per-host basis:

 - ocp_cluster_name
 - api_vip
 - dns_vip
 - ingress_vip
 - ovirt_cluster_name
 - ovirt_storage_domain_name
 - engine_url
 - engine_user
 - engine_password

Well, here's the playbook:

```yaml
- name: Deploy OCP cluster on oVirt
  hosts: primary_cluster
  remote_user: root
  tasks:
    # You also need to have a similar task in your playbook in order to obtain ovirt_auth token
    - name: Authenticate with oVirt engine
      ovirt_auth:
        url: "{{ engine_url }}"
        username: "{{ engine_user }}"
        password: "{{ engine_password }}"
        insecure: true

    - include_role:
        name: ovirt.ocp4-ipi
      vars:
        ocp_base_domain: ocptest.org
        pull_secret_file: /home/jan/ansible_dev/pull-secret.txt
        ocp_installer_version: 4.5
        ssh_keys:
          - "key1"
          - "key2"
```

After-installation tasks
------------------------

After a successful installation, if you want to use `openshift-install` command to manipulate your cluster
(e.g. to destroy it), you either need
- to export `OVIRT_CONFIG` variable and point it to `ovirt-config.yaml` in your installation directory or
- move the `ovirt-config.yaml` file from your installation directory to `/root/.ovirt/`

If you want to control your newly deployed OpenShift clsuter, the `oc` tool has been downloaded to your installation
directory.
In order to use it, just export `KUBECONFIG` variable and point it to `kubeconfig` file in `resources` directory.
For example:

```shell
export KUBECONFIG=/root/ocp4-ipi-secondary-20200422T163250/resources/auth/kubeconfig
```

Then you can use `oc` tool to communicate with your OpenShift cluster:

```shell
oc get clusteroperators
```

Contact information
------------------

Email: jzmeskal@redhat.com
