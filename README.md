# Migrating from VMware to Openstack: a practical guide with os-migrate

## The VMware price crisis and the path to Openstack

Broadcom's acquisition of VMware in 2023 fundamentally disrupted enterprise virtualization economics. Customers reported license price increases ranging from 800% to 1,500%, with some organizations facing demands for annual costs exceeding their entire infrastructure budgets. In response, enterprises pivoted toward open-source alternatives—**Proxmox** and **Nutanix** gained traction for traditional virtualization workloads. However, when organizations needed more than a hypervisor, when they required a complete cloud computing platform with multi-tenancy, self-service capabilities, and sophisticated orchestration, **Openstack** emerged as the strategic solution.

## os-migrate: beyond Openstack-to-Openstack

[**os-migrate**](https://github.com/os-migrate/os-migrate)  originated as a pragmatic toolkit for migrating workloads between Openstack clusters using ansible playbooks. Organizations running older Openstack versions could avoid the painful sequential upgrade path by deploying a fresh cluster and using os-migrate to transfer resources. Similarly, when infrastructure constraints required architectural changes—migrating from **linuxbridge** networking to modern **OVN**—os-migrate provided a non-disruptive pathway. Recognizing this success, the project naturally expanded to support a far more ambitious use case: **VMware-to-Openstack migration**.

## The technical challenges: resource mapping and driver compatibility

VMware-to-Openstack migration surfaces three critical architectural mismatches that os-migrate must navigate. **Resource allocation mapping** represents the first challenge: VMware's granular vSphere configurations must translate into Openstack's abstracted **flavor** system, where VMs are mapped to predefined or dynamically created hardware profiles. **Network topology translation** compounds this complexity—VMware's distributed switching and VLAN configurations must map to neutron networks, with security groups and port definitions recreated on the destination. Finally, **operating system driver compatibility** presents a hidden pitfall: VMware guests expect drivers like e1000 or vmxnet3, but Openstack's KVM hypervisor demands **VirtIO paravirtualized drivers** for optimal performance. Without these drivers pre-installed, migrated instances fail to boot or suffer severe I/O degradation.

## Three phases of VMware-to-Openstack migration

The migration process unfolds in three sequential phases, each serving a distinct purpose in the overall workflow.

### Discovery Phase

The **discovery phase** analyzes the source VMware environment without impacting running workloads. This phase gathers comprehensive metadata about each virtual machine, including CPU allocation, memory configuration, disk capacities and arrangements, network adapter details, and storage mappings. The collected data helps operators understand the migration scope and plan resource requirements in the destination Openstack cloud. Os-migrate uses the VMware community Ansible collections to connect to vCenter and retrieve this information through standard VMware APIs. The discovery output provides the foundation for subsequent migration planning decisions.

### Pre-migration Phase

The **pre-migration phase** ensures the destination Openstack cloud is properly configured and ready to receive migrated workloads. This phase validates network configurations in the destination environment, confirms or creates target networks matching the source topology, and most critically, deploys the **conversion host**—a specialized Openstack instance that serves as the migration engine. The conversion host is typically a RHEL 9.5 or CentOS 10 instance with sufficient resources (recommended 2GB RAM and 1 vCPU per concurrent migration). This phase also establishes network connectivity between the conversion host and the VMware environment, ensuring ports 443 and 902 are accessible for vCenter communication and VDDK operations.

### Migration Phase

The **migration phase** executes the actual workload transfer, with multiple workflow options to accommodate different infrastructure capabilities and downtime requirements. Operators can choose between three primary migration modes:

1. **NBDKit with CBT (Change Block Tracking)**: This mode enables near-zero downtime migrations by capturing an initial snapshot while the VM runs, then performing incremental synchronizations of only changed blocks. The final cutover involves a brief pause to transfer the last delta and perform the conversion. This approach is ideal for large disks where minimizing downtime is critical.

2. **Virt-v2v with conversion host**: This traditional approach uses the virt-v2v tool running on the conversion host to perform the migration. It can operate in single-pass mode (cold migration) or leverage CBT for multi-pass synchronization. The conversion host handles driver injection, ensuring migrated VMs have the necessary VirtIO drivers for KVM operation.

3. **Direct migration without conversion host**: This mode performs migration directly from a Linux machine, uploading converted volumes as Glance images or Cinder volumes. While eliminating the need for a conversion host, this approach is significantly slower and not recommended for large disk workloads or high-volume migrations.

During migration, os-migrate handles critical transformations including disk format conversion (VMDK to qcow2 or raw), network driver replacement (e1000/vmxnet3 to VirtIO), flavor mapping or creation, and network port provisioning with optional MAC address preservation.

## Environment setup: configuration in three files

Setting up os-migrate requires configuring three essential yaml files. First, install the os-migrate collection from ansible galaxy:

```bash
ansible-galaxy collection install os_migrate.vmware_migration_kit
```

### The inventory file

The inventory file defines the Ansible execution architecture using standard Ansible inventory format. It must specify the **migrator host** (typically localhost with local connection) and the **conversion host** with its access credentials. The migrator host runs the Ansible playbooks and orchestrates the migration workflow, while the conversion host performs the actual disk conversion and data transfer operations. 

Mandatory parameters include the conversion host IP address, SSH user (typically 'cloud-user' for RHEL/CentOS), and the private key file path for authentication. Optional parameters allow customization of SSH port, Python interpreter path, and Ansible connection settings. Additional configuration options are documented in the [os-migrate Variables Guide](https://os-migrate.github.io/documentation/), which covers advanced scenarios like using multiple conversion hosts for parallel migrations or attaching conversion hosts directly to public networks.

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

The vars.yaml file controls migration behavior and resource mapping logic. Mandatory parameters include `os_migrate_vmw_data_dir` (working directory for migration data, typically /opt/os-migrate), `vms_list` (array of VM names to migrate), and either `openstack_private_network` or network mapping definitions. 

Critical behavior flags include `already_deploy_conversion_host` (set to true to reuse an existing conversion host), `os_migrate_create_network_port` (controls whether os-migrate creates neutron ports before instance creation), and `use_existing_flavor` (determines flavor selection strategy). The file also supports advanced options like `security_groups` (UUIDs for instance security), `ssh_key_name` (keypair for instance access), `used_mapped_networks` (enables explicit network mapping), and `network_map` dictionary for translating VMware network names to Openstack network UUIDs. 

When running from an Ansible Execution Environment container, set `runner_from_aee: true` to prevent redundant dependency installation. The official documentation provides comprehensive variable references, including CBT-specific settings, conversion host sizing parameters, and parallel migration configurations.

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

The secrets.yaml file contains sensitive authentication credentials for both source and destination environments. For VMware connectivity, mandatory parameters include `esxi_hostname` or `vcenter_hostname` (vCenter FQDN or IP), `vcenter_username` (with sufficient privileges for snapshot creation and datastore access), `vcenter_password`, and `vcenter_datacenter` (datacenter name in vCenter inventory). 

The Openstack destination requires `dst_cloud` dictionary containing standard Keystone authentication parameters: `auth_url` (Keystone API endpoint), `username`, `password`, `project_name` or `project_id`, `user_domain_name`, `region_name`, `interface` (typically 'public'), and `identity_api_version` (should be 3). The `dst_cloud` structure can alternatively reference a clouds.yaml file by setting `copy_openstack_credentials_to_conv_host: true` and providing the cloud name. 

Best practices recommend storing this file securely with restrictive permissions (chmod 600) and using Ansible Vault encryption for production environments. Additional authentication options, including application credentials for SSO-enabled environments, are documented in the Openstack client configuration guides.

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

When executing the migration command, os-migrate processes the three migration phases sequentially with visible progress indicators.

Execute the migration with:

```bash
ansible-playbook -i inventory.yml os_migrate.vmware_migration_kit.migration \
  -e @secrets.yml -e @vars.yaml
```

### Discovery phase execution

The playbook begins by connecting to vCenter using the provided credentials and enumerating the VMs specified in the vms_list. For each VM, it retrieves configuration details including virtual hardware specifications, disk attachments, network adapters, and snapshot states. The output includes validation messages confirming successful VMware API connectivity and summary statistics of discovered VMs. This phase typically completes in seconds to minutes depending on the number of VMs and vCenter responsiveness.

### Pre-migration phase execution

The pre-migration phase validates Openstack authentication by connecting to Keystone and confirming project access. If `already_deploy_conversion_host` is false, os-migrate provisions a new conversion host instance, creates necessary network infrastructure (unless disabled), assigns a floating IP, and installs required packages (nbdkit, VDDK libraries, virt-v2v). The playbook then verifies network connectivity between the conversion host and VMware environment, testing DNS resolution of the vCenter hostname and port accessibility (443 and 902). Progress output includes Ansible task status for conversion host deployment, network validation results, and confirmation of successful SSH connectivity to the conversion host.

### Migration phase execution

The migration phase iterates through each VM in the vms_list, displaying real-time progress for disk data transfer. For NBDKit migrations, the output shows the nbdkit server startup, NBD connection establishment, and transfer progress with percentage completion and data rates. When CBT is enabled, the first pass completes the bulk transfer while the VM remains running, followed by incremental sync passes showing only changed blocks. The final cutover triggers a VM snapshot, transfers the delta, and initiates the conversion process. For virt-v2v migrations, progress includes conversion pod creation, driver injection status, and bootloader configuration updates. The playbook reports instance creation in Openstack with assigned IPs and completes with a summary showing successful migrations, failures, and total execution time. Logs are available on the conversion host in /tmp/osm-nbdkit-<vm-name>-<random-id>.log for detailed troubleshooting if issues occur.

## Choosing your migration strategy: performance trade-offs

The choice between **cold migration** and **warm migration** fundamentally affects both total execution time and downtime. Cold migration takes VMs offline, captures full disk snapshots, performs conversion, and boots instances—straightforward but downtime-intensive. Warm migration leverages **Change Block Tracking (CBT)**, capturing initial snapshots while VMs run, then incrementally syncing only changed blocks before a brief final cutover. The performance difference is dramatic and [shown below](https://os-migrate.github.io/documentation/#_table_2):

| Disk Size | CBT Enabled | Total Time | Cutover Time | Expected Downtime |
|-----------|-------------|------------|--------------|-------------------|
| 100 GB    | Yes         | 8 min      | 2 min        | 2 min             |
| 100 GB    | No          | 7 min      |        | 7 min             |
| 1 TB      | Yes         | 39 min     | 2 min        | 2 min             |
| 1 TB      | No          | 35 min     |        | 35 min            |

For parallel migrations, performance scales predictably—a single conversion host handling 6 concurrent threads migrates 20 VMs in just 15 minutes versus 31 minutes with sequential execution.

## Conclusion

The path from VMware's punitive pricing to Openstack's open architecture is now clearly marked. With os-migrate, the technical hurdles of cross-hypervisor migration transform from a daunting obstacle into a manageable, well-documented process that enterprises can execute with confidence.
