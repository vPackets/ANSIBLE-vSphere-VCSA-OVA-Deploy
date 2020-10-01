# ANSIBLE-vSphere-VCSA-OVA-Deploy


## Using Ansible to deploy a vSphere 7.0 vCenter automatically
by Nicolas MICHEL [@vpackets](https://twitter.com/vpackets) / [LinkedIn](https://www.linkedin.com/in/mclnicolas/) / [Blog](http://vpackets.net/) 

_**This disclaimer informs readers that the views, thoughts, and opinions expressed in this series of posts belong solely to the author, and not necessarily to the author’s employer, organization, committee or other group or individual.**_

### Introduction ###

I had a need to automate my NSX-T lab deployment. I could have used a single tool to perform all the relevant task but I wanted to learn and will use multiple scripts and tools.
This particular repository uses Ansible and will run the following tasks:

- Deploy the VCSA OVA into an ESXi host.
- Configure basic settings on the vCenter.
- Add hosts and Licenses to the vCenter.

The Ansible module vmware_dvswitch does not support the creation of virtual switch based on the 7.0.0 version. I will use powercli or another tool for that specific use case.

### Prerequisites ###

I have used [my own docker container](https://hub.docker.com/repository/docker/vpackets/tools) - ([Source code](https://github.com/vPackets/DOCKER-Tools))  to run this particular repository.
It contains all the libraries and software I need to accomplish my day to day job.

I am using Ansible version 2.9.10 and Pyvmomi 7.0.

a DNS server must be able to resolve DNS queries for all the fqdn in your labs.


### Version Used ###

```
Ubuntu Version :        18.04.3 LTS 
Ansible:                2.9.10
Pyvmomi:                7.0
VCSA OVA:               7.0.0.10300-16189094
```


### Software Requirements #

It is necessary to decompress the VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ova.

```
Step 1: Rename VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ova to VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.tgz
Step 2: tar xvf VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.tgz 

x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ovf
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.mf
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.cert
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-file1.json
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-file2.rpm
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk1.vmdk
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk2.vmdk
x VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk3.vmdk
```



The following files will be created and needed to deploy the OVF to the ESXi host.
```
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk1.vmdk
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk2.vmdk
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-disk3.vmdk
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-file1.json
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10-file2.rpm
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.cert
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.mf
├── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ova
└── VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ovf
```



### Code architecture ###

```
|-- README.md
|-- answerfile.yml
|-- deploy.yml
|-- playbooks
|   |-- 00-deploy-vcsa.yml
|   |-- 01-basic-config-vcsa.yml
|   |-- 02-add-hosts.yml
|   |-- 03-storage.yml
|   `-- 99-add-networking.yml
`-- vcenter-properties.yml
```

I wanted to have something flexible so that I could test a single playbook without impacting the code on other playbooks.
Deploy.yml is the main playbook to run. It will call each playbooks in order.

```
- import_playbook: playbooks/00-deploy-vcsa.yml
- import_playbook: playbooks/01-basic-config-vcsa.yml
- import_playbook: playbooks/02-add-hosts.yml
- import_playbook: playbooks/03-storage.yml
```


#### Variables ####

You can change the following variable based on your needs:

```
esxi_username: 'root'
esxi_ip_address: '172.31.100.10'
esxi_infrastructure_fqdn: 'srv-esxi-01.megasp.net'
esxi_compute_01_fqdn: 'srv-esxi-02.megasp.net'
esxi_compute_02_fqdn: 'srv-esxi-03.megasp.net'
vcenter_username: 'administrator@megasp.net'
vcenter_virtual_machine_name: 'SRV-VCENTER-01'
vcenter_hostname: 'srv-vcenter-01'
vcenter_net_ip_address: '172.31.100.5'
vcenter_net_prefix: '24'
vcenter_net_gateway: '172.31.100.1'
vcenter_dns_servers: '172.31.100.20'
domain: 'megasp.net'
searchpath: ""
vcsa_size: 'tiny'
vcsa_ova_file: '/home/nmichel/lab-images/VMware-vCenter-Server-Appliance-7.0.0.10300-16189094_OVF10.ova'
datastore: 'Synology-NFS-Datastore'
esxi_network: 'VM Network'
datacenter_name: 'ATX-01'
cluster_name_edge: 'Management-Edge'
cluster_name_compute: 'Compute'
host_folder_name_edge: 'ATX-01-EDGE'
host_folder_name_compute: 'ATX-01-COMPUTE'
vm_folder_name_infrastructure: 'DC-INFRASTRUCTURE'
vm_folder_name_nsx_infrastructure: 'NSX-INFRASTRUCTURE'
vm_folder_name_tenant_01: 'COMPUTE-TENANT-01'
vm_folder_name_tenant_02: 'COMPUTE-TENANT-02'
```

Some variables will be prompted and remain private.

```
What is the ESXi Password ?: 
What is the vCenter Password ?: 
What is the License for vCenter ?: 
What is the License for vSphere7 ?: 
```



#### Playbook Execution ####

```
nmichel@laptop:/Users/nmichel/vcsa-deploy/ansible-playbook
PLAY [Lab Deployment] ***************************************************************************************************************************************************************************************************************************************************************************************************************************

TASK [setting fact] *****************************************************************************************************************************************************************************************************************************************************************************************************************************
ok: [localhost]

PLAY [Deploy the VCSA to an ESXi Host] **********************************************************************************************************************************************************************************************************************************************************************************************************

TASK [vmware_deploy_ovf] ************************************************************************************************************************************************************************************************************************************************************************************************************************
[WARNING]: Problem validating OVF import spec: Line 764: Invalid value 'Network 1' for element 'Connection'.
changed: [localhost]

TASK [pause] ************************************************************************************************************************************************************************************************************************************************************************************************************************************
Pausing for 2400 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [localhost]

PLAY [Basic VCSA Configuration] *****************************************************************************************************************************************************************************************************************************************************************************************************************

TASK [Provide information about vCenter] ********************************************************************************************************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Create Datacenter] ************************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create Cluster - Management - EDGE] *******************************************************************************************************************************************************************************************************************************************************************************************************
[DEPRECATION WARNING]: Configuring HA using vmware_cluster module is deprecated and will be removed in version 2.12. Please use vmware_cluster_ha module for the new functionality.. This feature will be removed in version 2.12. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
[DEPRECATION WARNING]: Configuring DRS using vmware_cluster module is deprecated and will be removed in version 2.12. Please use vmware_cluster_drs module for the new functionality.. This feature will be removed in version 2.12. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
changed: [localhost]

TASK [Create Cluster - Compute] *****************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create Host Folder - EDGE] ****************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create Host Folder - COMPUTE] *************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create VM Folder - INFRASTRUCTURE] ********************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create VM Folder - NSX INFRASTRUCTURE] ****************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create VM Folder - TENANT 01] *************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create VM Folder - TENANT 02] *************************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

PLAY [Add Hosts and licenses to vCenter] ********************************************************************************************************************************************************************************************************************************************************************************************************

TASK [Add ESXi Host to vCenter - EDGE] **********************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Add ESXi Host to vCenter - COMPUTE 01] ****************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Add ESXi Host to vCenter - COMPUTE 02] ****************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Add a new vCenter license - vCenter] ******************************************************************************************************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Add ESXi license and assign to the ESXi host - EDGE] **************************************************************************************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Add ESXi license and assign to the ESXi host - COMPUTE 01] ********************************************************************************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Add ESXi license and assign to the ESXi host - COMPUTE 02] ********************************************************************************************************************************************************************************************************************************************************************************
ok: [localhost]

PLAY RECAP **************************************************************************************************************************************************************************************************************************************************************************************************************************************
localhost                  : ok=20   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```



#### TO DO ####

    - Host folder : Move the Cluster compute and Edge into their respective folder ATX 