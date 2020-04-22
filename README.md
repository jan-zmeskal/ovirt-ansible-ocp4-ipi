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

Additional tasks
----------------

The role also performs some quality-of-life related tasks. Overwrite the following variables to determine if they
should be executed or not.

| Name | Default value | Description |
|------|---------------|-------------|
| perform_dns_check | true | # If set to true, role will fail early if you haven't set up DNS entries for API and Ingress |
| include_engine_ssh_key | true  | If set to true, public SSH key of engine machine will be included in RHCOS nodes. It will be also generated, if it doesn't exist. |

Known issue
------------

With OpenShift4.4 there is a known bug, see here: https://bugzilla.redhat.com/show_bug.cgi?id=1818577.
`openshift-install` creates a default installation template that has some problems.
To overcome these problems, you may upload your custom [RHCOS](https://www.openshift.com/learn/coreos/)
template using [oVirt.image-template](https://github.com/oVirt/ovirt-ansible-image-template) role.
If you do, just pass the name of your custom template to `custom_rhcos_template` variable.
That will make OpenShift installer aware of your custom template.

Example Playbook
----------------

In my testing, I'm using the following playbook. Keep in mind though that most of the playbook is actually dedicated
to uploading custom RedHat CoreOS image to oVirt engine and making an oVirt template out of it. Unless you specifically
want to do that, you need just two tasks in your playbook - authenticate with oVirt and include this role.

Also, you might notice that some of the required variables are missing here. That's because I specify them on per-host
basis in my inventory file. Specifically, I set following variables on per-host basis:

 - ocp_cluster_name
 - api_vip
 - dns_vip
 - ingress_vip
 - ovirt_cluster_name
 - ovirt_storage_domain_name
 - rhcos_node_memory_mb
 - rhcos_node_cpu
 - engine_url
 - engine_user
 - engine_password

Well, here's the playbook:

    - name: Deploy OCP cluster on oVirt
      hosts: secondary
      remote_user: root
      vars:
        # Following variables relate to uploading custom RHCOS image and are not required for running the role itself
        rhcos_downloaded_image: /root/rhcos.qcow2.gz
        rhcos_extracted_image: /root/rhcos.qcow2
        upload_custom_rhcos_image: true
        rhcos_template_name: rhcos_custom
        run_ocp_ipi: true  # This will actually execute ovirt-ansible-ocp4-ipi role in my playbook
      tasks:
        # You also need to have a similar task in your playbook in order to obtain ovirt_auth token
        - name: Authenticate with oVirt engine
          ovirt_auth:
            url: "{{ engine_url }}"
            username: "{{ engine_user }}"
            password: "{{ engine_password }}"
            insecure: true

        # The whole following block uploads RHCOS image to oVirt engine and customizes it
        # I do that for testing purposes. If you are okay with the default template created by OpenShift installer,
        # you don't need to do any of that.
        - name: Upload custom RHCOS image to oVirt engine
          block:
            - name: Load rhcos.json from openshift/installer GitHub repository
              uri:
                url: "https://raw.githubusercontent.com/openshift/installer/release-4.4/data/data/rhcos.json"
              register: rhcos_json
              no_log: true

            - name: Download compressed RHCOS image
              get_url:
                url: "{{ rhcos_json.json.baseURI + rhcos_json.json.images.openstack.path }}"
                dest: "{{ rhcos_downloaded_image }}"
                force: no

            - name: Check if extracted image already exists
              stat:
                get_attributes: no
                get_checksum: no
                get_mime: no
                path: "{{ rhcos_extracted_image }}"
              register: stat_result

            - name: Extract downloaded RHCOS image
              shell: "gunzip -c {{ rhcos_downloaded_image }} > {{ rhcos_extracted_image }}"
              when: not stat_result.stat.exists

            - include_role:
                name: oVirt.image-template
              vars:
                # This is required so that oVirt.image-template does not download the RHCOS image again
                qcow_url: "file://{{ rhcos_extracted_image }}"
                image_path: "{{ rhcos_extracted_image }}"
                # Here I'm customizing the RHCOS template
                template_disk_storage: "{{ ovirt_storage_domain_name }}"
                template_cluster: "{{ ovirt_cluster_name }}"
                template_name: "{{ rhcos_template_name }}"
                template_cpu: "{{ rhcos_node_cpu }}"
                template_memory: "{{ rhcos_node_memory_mb }}MiB"
                template_memory_guaranteed: "8GiB"  # Here because of https://bugzilla.redhat.com/show_bug.cgi?id=1821215
                template_disk_size: "32GiB"  # Here because of https://bugzilla.redhat.com/show_bug.cgi?id=1818577
                template_disk_interface: virtio_scsi
                template_seal: no  # This is actually required for successful template creation
          when: upload_custom_rhcos_image

        - include_role:
            name: ovirt.ocp4-ipi
          vars:
            ocp_base_domain: ocptest.org
            pull_secret_file: /home/jan/ansible_dev/pull-secret.txt
            ssh_keys:
              - "SSH key of one user"
              - "SSH key of another user"
            custom_rhcos_template: "{{ rhcos_template_name }}"  # Omit this if you don't need to use custom RHCOS template
          when: run_ocp_ipi


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
