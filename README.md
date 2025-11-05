# Migrating from VMware to Openstack: a practical guide with os-migrate

## The VMware price crisis and the path to Openstack

Broadcom's acquisition of VMware in 2023 fundamentally disrupted enterprise virtualization economics. Customers reported license price increases ranging from 800% to 1,500%, with some organizations facing demands for annual costs exceeding their entire infrastructure budgets. In response, enterprises pivoted toward open-source alternatives—**Proxmox** and **Nutanix** gained traction for traditional virtualization workloads. However, when organizations needed more than a hypervisor, when they required a complete cloud computing platform with multi-tenancy, self-service capabilities, and sophisticated orchestration, **OpenStack** emerged as the strategic solution.

## os-migrate: beyond Openstack-to-Openstack

[**os-migrate**](https://github.com/os-migrate/os-migrate)  originated as a pragmatic toolkit for migrating workloads between OpenStack clusters using ansible playbooks. Organizations running older OpenStack versions could avoid the painful sequential upgrade path by deploying a fresh cluster and using os-migrate to transfer resources. Similarly, when infrastructure constraints required architectural changes—migrating from **linuxbridge** networking to modern **OVN**—os-migrate provided a non-disruptive pathway. Recognizing this success, the project naturally expanded to support a far more ambitious use case: **VMware-to-Openstack migration**.

## The technical challenges: resource mapping and driver compatibility

VMware-to-Openstack migration surfaces three critical architectural mismatches that os-migrate must navigate. **Resource allocation mapping** represents the first challenge: VMware's granular vSphere configurations must translate into OpenStack's abstracted **flavor** system, where VMs are mapped to predefined or dynamically created hardware profiles. **Network topology translation** compounds this complexity—VMware's distributed switching and VLAN configurations must map to neutron networks, with security groups and port definitions recreated on the destination. Finally, **operating system driver compatibility** presents a hidden pitfall: VMware guests expect drivers like e1000 or vmxnet3, but Openstack's KVM hypervisor demands **VirtIO paravirtualized drivers** for optimal performance. Without these drivers pre-installed, migrated instances fail to boot or suffer severe I/O degradation.

## Three phases of VMware-to-Openstack migration

The migration process unfolds in three sequential phases. The **discovery phase** analyzes the source VMware environment, gathering comprehensive metadata about each VM—CPU, memory, disk configurations, networks, and storage arrangements—without impacting running workloads. The **pre-migration phase** ensures destination readiness: validating network configurations, confirming or creating target networks, and deploying the **conversion host**, a critical Openstack instance that serves as the transient migration engine. Finally, the **migration phase** executes the actual workload transfer, adapting to infrastructure capabilities and downtime tolerances.

## Environment setup: configuration in three files

Setting up os-migrate requires configuring three essential yaml files. First, install the os-migrate collection from ansible galaxy:

```bash
ansible-galaxy collection install os_migrate.vmware_migration_kit
```

### The inventory file

Define your migrator (the machine running playbooks) and conversion host (the Openstack instance performing migration):

```yaml
migrator:
  hosts:
    localhost:
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"
conversion_host:
  hosts:
    192.168.18.205:
      ansible_ssh_user: cloud-user
      ansible_ssh_private_key_file: key
```

### The vars.yaml file

Controls migration behavior—whether to reuse an existing conversion host, network mappings, and the VM list:

```yaml
os_migrate_vmw_data_dir: /opt/os-migrate
already_deploy_conversion_host: true
openstack_private_network: 81cc01d2-5e47-4fad-b387-32686ec71fa4
security_groups: ab7e2b1a-b9d3-4d31-9d2a-bab63f823243
ssh_key_name: default
os_migrate_create_network_port: true
vms_list:
  - rhel-9.4-1
  - ubuntu-20.04-web
```

### The secrets.yaml file

Contains VMware vCenter credentials and Openstack authentication:

```yaml
esxi_hostname: 10.0.0.7
vcenter_hostname: 10.0.0.7
vcenter_username: administrator
vcenter_password: SecurePassword
vcenter_datacenter: Datacenter

dst_cloud:
  auth:
    auth_url: https://keystone-openstack.cloud:13000/v3
    username: admin
    project_name: migration
    password: openstack_password
    user_domain_name: Default
  region_name: regionOne
  interface: public
  identity_api_version: 3
```

### Running the migration

Execute the migration with:

```bash
ansible-playbook -i inventory.yml os_migrate.vmware_migration_kit.migration \
  -e @secrets.yml -e @vars.yaml
```

## Choosing your migration strategy: performance trade-offs

The choice between **cold migration** and **warm migration** fundamentally affects both total execution time and downtime. Cold migration takes VMs offline, captures full disk snapshots, performs conversion, and boots instances—straightforward but downtime-intensive. Warm migration leverages **Change Block Tracking (CBT)**, capturing initial snapshots while VMs run, then incrementally syncing only changed blocks before a brief final cutover. The performance difference is dramatic:

| Disk Size | CBT Enabled | Total Time | Cutover Time | Expected Downtime |
|-----------|-------------|------------|--------------|-------------------|
| 100 GB    | Yes         | 8 min      | 2 min        | 2 min             |
| 100 GB    | No          | 7 min      | 7 min        | 7 min             |
| 1 TB      | Yes         | 39 min     | 2 min        | 2 min             |
| 1 TB      | No          | 35 min     | 35 min       | 35 min            |

For parallel migrations, performance scales predictably—a single conversion host handling 6 concurrent threads migrates 20 VMs in just 15 minutes versus 31 minutes with sequential execution.

## Conclusion

The path from VMware's punitive pricing to Openstack's open architecture is now clearly marked. With os-migrate, the technical hurdles of cross-hypervisor migration transform from a daunting obstacle into a manageable, well-documented process that enterprises can execute with confidence.
