# Configuring vSphere Permissions Using Ansible

This directory contains an Ansible Playbook that can be used to configure vSphere permissions for any purpose (e.g. TKG, NSX ALB, Kasten, etc).
This is necessary when least privilege is desired, rather than specifiying an administrative account.

This directory has pre-defined examples for TKG, NSX ALB, and Kasten.

Using the `tkg-vars.yml` variables file will perform the following:

- Creates 3 vSphere roles with the necessary privileges:
  - Role for TKG at the vCenter level
  - Role for NSX ALB at the vCenter level
  - Role for NSX ALB at the Service Engine folder level
- Applies the global TKG role at the vCenter level, using the TKG service account
- Applies the global NSX ALB roles at the vCenter level, using the NSX ALB service account
- Applies the folder NSX ALB role at the Service Engine folder level, using the NSX ALB service account

Using the `k10-vars.yml` variables file will create a vSphere role and bind it to the Kasten service account at the vCenter level.

**Prerequisites:**

For TKG/NSX ALB:

- Service account for TKG (e.g `tkg-admin@terasky.demo`). It can also be a `vsphere.local` user if desired.
- Service account for NSX ALB (e.g `tkg-nsxalb-admin@terasky.demo`). It can also be a `vsphere.local` user if desired.
- vSphere VM folder for the NSX ALB Service Engines (e.g. `nsxalb_service_engines`).

For Kasten:

- Service account for Kasten (e.g. `tkg-k10@terasky.demo`). It can also be a `vsphere.local` user if desired.

## Ansible Playbook Usage

Install the required Ansible collection using the `requirements.yml` file.

```bash
ansible-galaxy install -r requirements.yml
```

Install the required Python modules using the `requirements.txt` file.

```bash
pip3 install -r requirements.txt
```

Update the variables `vars.yml` file.

For TKG/NSX ALB:

Update the `tkg-vars.yml` file using your configuration.
Parameters you must update:

- `vcenter_hostname`
- `vcenter_username`
- `vcenter_password`
- Under `vsphere_roles`, specify the appropriate `user` for all roles in the array, using your domain and service accounts.
- For the `tkg-nsxalb-folder` role, specify your NSX ALB Service Engines VM folder name, unless you are using the same name as ours.

For Kasten:

Update the `k10-vars.yml` file using your configuration.
Parameters you must update:

- `vcenter_hostname`
- `vcenter_username`
- `vcenter_password`
- Under `vsphere_roles`, specify the appropriate `user` for all roles in the array, using your domain and service accounts.

You can use this playbook to either view or configure the vSphere roles.

To configure the vSphere roles, run:

```bash
# TKG
ansible-playbook playbook.yml -e action=configure -e=@tkg-vars.yml

# Kasten
ansible-playbook playbook.yml -e action=configure -e=@k10-vars.yml
```

To view the vSphere roles, run:

```bash
# TKG
ansible-playbook playbook.yml -e action=view -e=@tkg-vars.yml

# Kasten
ansible-playbook playbook.yml -e action=view -e=@k10-vars.yml
```

>The `view` option can be extremely useful when troubleshooting permissions, reproducing the privileges or simply for retrieving/extracting the IDs of the privileges of a role.

___

Reference:

- [Ansible Collection: community.vmware](https://github.com/ansible-collections/community.vmware)
- [TKG - Required Permissions vSphere](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.5/vmware-tanzu-kubernetes-grid-15/GUID-mgmt-clusters-vsphere.html#vsphere-permissions)
- [NSX ALB - Roles and Permissions for vSphere](https://avinetworks.com/docs/21.1/roles-and-permissions-for-vcenter-nsx-t-users/)
- <https://avinetworks.com/docs/latest/vmware-user-role>
