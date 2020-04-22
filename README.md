ovirt-ansible-ocp4-ipi
=========

> :warning: This role is not officially supported by Red Hat. It's my personal project to ease my daily work.

This role automates deployment of OpenShift 4 IPI (installer-provided infrastructure) on top of oVirt.
oVirt is supported as an OpenShift platform (or provider) since OpenShift 4.4.
The IPI installation using `openshift-install` is admittedly quite easy
in itself and requires only minimal manual intervention.
Therefore this role is mainly intended for those who needs to perform it repeatedly,
possibly on many oVirt engines (quality engineering, performance testing etc.)

Requirements
------------

- oVirt >= 4.3.9

This role is intended to be executed against your oVirt engine.
As long as you use oVirt engine as the target hosts, all of the below requirements should be in place:

- Python SDK version >= 4.3
- Ansible >= 2.9

You also need to authenticate with oVirt engine before executing this role.
Use ansible module [ovirt_auth](https://docs.ansible.com/ansible/latest/modules/ovirt_auth_module.html) to do that.

Required Variables
--------------

| Name | Default value | Description |
|------|---------------|-------------|
| ocp_base_domain | UNDEF | DNS domain of your OCP cluster. |
| ocp_cluster_name | UNDEF | The name of your OCP cluster. This name will be included in the resulting domain of apps running on OCP like this: `console-openshift-console.apps.<ocp_cluster_name>.<ocp_base_domain>` |



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

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }


Contact Information
------------------

Email: jzmeskal@redhat.com
