na_ontap_ai_aff_deploy
=========

Ansible role that can be used to deploy/provision a new NetApp AFF/FAS cluster as part of an ONTAP AI converged infrastructure pod.

Requirements
------------

**Prerequesites:**

1. This role assumes the storage system is an AFF A800 with X1146 or X1148 network cards in slots 3 and 5 in each controller (2x cards/controller). 
2. Storage system nodes should be cabled as specified in [NVA-1151](https://www.netapp.com/pdf.html?item=/media/20708-nva-1151-deploy.pdf). 
3. Initial cluster setup/creation must have already been completed.

If the hardware configuration is differen than above the disk and/or network port configuration variables must be modified. 

Role Variables
--------------

By default, this role accommodates an AFF A800 cluster containing two nodes, identified as 'ontap-01' and 'ontap-02', and creates the storage configuration defined in [NVA-1151](https://www.netapp.com/pdf.html?item=/media/20708-nva-1151-deploy.pdf).
To override the default values, create a new variables file using the example below as a template and include it in the playbook used to execute this role. An example of this is shown
in the Example Playbook section below.

- If there are more than two nodes in the cluster add additional variable entries for each cluster node.  
- If the storage system is not an A800 modify the disk and network port configurations accordingly. 
- To create volumes for production use modify the default volume configuration as needed. 

The required variables and their values as defined in defaults/main.yml are listed below-

```
---
# defaults file for na_ontap_ai_aff_deploy

# ONTAP cluster management hostname/IP
netapp_hostname: "192.168.100.100"

# Cluster admin login details
netapp_username: admin
netapp_password: password

# Cluster SVM details
cluster_svm: cluster_svm

# Data SVM details
data_svm: "data_01"
data_svm_rootvol: "{{data_svm}}_rootvol"
allowed_protocols: nfs

# Storage VLAN details
storage_vlans:
  - {desc: nfs_vlan_01, id: 3111}

# Broadcast domain details
broadcast_domains:
  nfs_vlan_01: "NFS-A"

# FlexVol volumes to be created as part of deployment
flexvols:
  - {vname: testvol01, aggr: "n01_data01", size: "100", size_unit: gb, jpath: /testvol01}

# FlexGroup Volumes to be created as part of deployment
flexgroups:
  - {vname: data_flexgroup_01, size: "20", size_unit: tb, jpath: /data_flexgroup_01}

# New export policy details (to be created as part of deployment)
create_new_export_policy: False
new_export_policy_name: ontapai

# Export policy rule details (to be created as part of deployment for both the default and new export policies)
exp_rules:
  - {clientmatch: 192.168.0.0/24, rorule: sys, rwrule: sys, superuser: sys, protocol: nfs }

# Determines whether or not Harvest role and user are to be created
create_harvest_role_user: True
harvest_role_cmds:
  - "version"
  - "cluster identity show"
  - "cluster show"
  - "system node show"
  - "statistics"
  - "lun show"
  - "network interface show"
  - "qos workload show"

# Harvest role password
harvest_password: "NetApp!23"

# Cluster node details
node1_name: "ontap-01"
node2_name: "ontap-02"
nodes:
  - {name: "{{ node1_name }}", ifgrp1: a11a, ifgrp2: a21a, data_aggr: "n01_data01", aggr_disk_count: 47}
  - {name: "{{ node2_name }}", ifgrp1: a12a, ifgrp2: a22a, data_aggr: "n02_data01", aggr_disk_count: 47}

# Determines whether or not data aggregrates are to be created
create_data_aggrs: True

# Network ports used. These will be removed from the default broadcast domain.
ports: "{{ node1_name }}:e3a,{{ node1_name }}:e3b,{{ node1_name }}:e5a,{{ node1_name }}:e5b,{{ node2_name }}:e3a,{{ node2_name }}:e3b,{{ node2_name }}:e5a,{{ node2_name }}:e5b"

# LACP interface group details
ifgrps:
  - {grpname: a11a, node: "{{ node1_name }}", ports: [e3a, e5a], mtu: 9000}
  - {grpname: a21a, node: "{{ node1_name }}", ports: [e3b, e5b], mtu: 9000}
  - {grpname: a12a, node: "{{ node2_name }}", ports: [e3a, e5a], mtu: 9000}
  - {grpname: a22a, node: "{{ node2_name }}", ports: [e3b, e5b], mtu: 9000}

# VLAN 3111 port details
ports_3111: "{{ node1_name }}:a11a-3111,{{ node1_name }}:a21a-3111,{{ node2_name }}:a12a-3111,{{ node2_name }}:a22a-3111"

# Data interface/LIF details
interfaces:
  - {iname: "data01-n01-a", homeport: a11a-3111, homenode: "{{ node1_name }}", address: 192.168.0.101, netmask: 255.255.255.0}
  - {iname: "data01-n01-b", homeport: a11a-3111, homenode: "{{ node1_name }}", address: 192.168.0.102, netmask: 255.255.255.0}
  - {iname: "data01-n01-c", homeport: a21a-3111, homenode: "{{ node1_name }}", address: 192.168.0.103, netmask: 255.255.255.0}
  - {iname: "data01-n01-d", homeport: a21a-3111, homenode: "{{ node1_name }}", address: 192.168.0.104, netmask: 255.255.255.0}
  - {iname: "data01-n02-a", homeport: a12a-3111, homenode: "{{ node2_name }}", address: 192.168.0.105, netmask: 255.255.255.0}
  - {iname: "data01-n02-b", homeport: a12a-3111, homenode: "{{ node2_name }}", address: 192.168.0.106, netmask: 255.255.255.0}
  - {iname: "data01-n02-c", homeport: a22a-3111, homenode: "{{ node2_name }}", address: 192.168.0.107, netmask: 255.255.255.0}
  - {iname: "data01-n02-d", homeport: a22a-3111, homenode: "{{ node2_name }}", address: 192.168.0.108, netmask: 255.255.255.0}
```

Dependencies
------------

N/A

Example Playbooks
----------------

Note: Due to the nature of NetApp Ansible modules, this role must be invoked against "localhost". The NetApp modules will handle communication with the ONTAP cluster.
The cluster IP address and admin user credentials must be specified in either defaults/main.yml or a variable override file included in the playbook as shown in the example below.

    - name: "ONTAP AI: Deploy/provision ONTAP/AFF Cluster"
      hosts: localhost
      remote_user: root
      gather_facts: false
      vars_files:
        - <vars_file>
      roles:
         - role: na_ontap_ai_aff_deploy

License
-------

BSD
