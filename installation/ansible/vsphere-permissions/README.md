# Configuring vSphere Permissions Using Ansible

This directory contains an Ansible Playbook that can be used to configure vSphere permissions for any purpose.
This is necessary when least privilege is desired, rather than specifiying an administrative account.

This directory has a pre-defined example for Kasten.

Using the `k10-vars.yml` variables file will create a vSphere role and bind it to the Kasten service account at the vCenter level.

**Prerequisites:**

Service account for Kasten (e.g. `tkg-k10@terasky.demo`). It can also be a `vsphere.local` user if desired.

## Ansible Playbook Usage

Install the required Ansible collection using the `requirements.yml` file.

```bash
ansible-galaxy install -r requirements.yml
```

Install the required Python modules using the `requirements.txt` file.

```bash
pip3 install -r requirements.txt
```

Update the `k10-vars.yml` file using your configuration.
Parameters you must update:

- `vcenter_hostname`
- `vcenter_username`
- `vcenter_password`
- Under `vsphere_roles`, specify the appropriate `user` for all roles in the array, using your domain and service accounts.

You can use this playbook to either view or configure the vSphere roles.

To configure the vSphere roles, run:

```bash
ansible-playbook playbook.yml -e action=configure -e=@k10-vars.yml
```

To view the vSphere roles, run:

```bash
ansible-playbook playbook.yml -e action=view -e=@k10-vars.yml
```

>The `view` option can be extremely useful when troubleshooting permissions, reproducing the privileges or simply for retrieving/extracting the IDs of the privileges of a role.
