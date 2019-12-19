# Creating the test infrastructure
The following steps create the virtual test environment infrastructure including VMs, libvirt networks and setting up the virtual BMCs.
The same steps can be achieved from the [ocp-edge-demo-virt job](https://jenkins-fci-continuous-productization.cloud.paas.psi.redhat.com/job/ocp-edge-demo-virt/) by selecting the Provision stage only, without running the Deploy stage.

```bash
[root@titan47 ~]# mkdir ocp-edge-virt-env
[root@titan47 ~]# cd ocp-edge-virt-env/
[root@titan47 ~]# yum install -y libvirt-devel python3-virtualenv gcc git libvirt-python3
[root@titan47 ~]#dnf localinstall -y http://download.eng.bos.redhat.com/brewroot/vol/rhel-8/packages/sshpass/1.06/3.el8ae/x86_64/sshpass-1.06-3.el8ae.x86_64.rpm
[root@titan47 ocp-edge-virt-env]# virtualenv virtualenv
[root@titan47 ocp-edge-virt-env]# source virtualenv/bin/activate
(virtualenv) [root@titan47 ocp-edge-virt-env]# pip install ansible==2.8 linchpin libvirt-python netaddr lxml
(virtualenv) [root@titan47 ocp-edge-virt-env]# curl -o virtualenv/bin/install_selinux_venv.sh https://raw.githubusercontent.com/CentOS-PaaS-SIG/linchpin/1792bb8fa02c4acbef63c987f715f0c3cf8b193e/scripts/install_selinux_venv.sh; bash -x virtualenv/bin/install_selinux_venv.sh 
(virtualenv) [root@titan47 ocp-edge-virt-env]# git -c http.sslVerify=false clone https://gitlab.cee.redhat.com/ocp-edge-qe/ocp-edge-demo.git
(virtualenv) [root@titan47 ocp-edge-virt-env]# cd ocp-edge-demo/linchpin-workspace/
## Edit custom vars in hooks/ansible/ocp-edge-setup/extravars.yaml if needed
(virtualenv) [root@titan47 linchpin-workspace]# ansible -c local localhost -m template -a "src=hooks/ansible/ocp-edge-setup/extravars.yaml dest=$PWD/extravars.yaml" -e @hooks/ansible/ocp-edge-setup/extravars.yaml
(virtualenv) [root@titan47 linchpin-workspace]# linchpin --template-data @extravars.yaml -v destroy libvirt-network libvirt-new
(virtualenv) [root@titan47 linchpin-workspace]# linchpin --template-data @extravars.yaml -v up libvirt-network libvirt-new cfgs

(virtualenv) [root@titan47 linchpin-workspace]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 14    worker-1                       running
 -     master-0                       shut off
 -     master-1                       shut off
 -     master-2                       shut off
 -     worker-0                       shut off

(virtualenv) [root@titan47 linchpin-workspace]# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 baremetal            active     no            yes
 default              active     yes           yes
 provisioning         active     no            yes

[root@titan47 ~]# /root/.virtualenvs/vbmc/bin/python /root/.virtualenvs/vbmc/bin/vbmc list
+-------------+---------+----------------------+------+
| Domain name | Status  | Address              | Port |
+-------------+---------+----------------------+------+
| master-0    | running | ::ffff:192.168.123.1 | 6230 |
| master-1    | running | ::ffff:192.168.123.1 | 6231 |
| master-2    | running | ::ffff:192.168.123.1 | 6232 |
| worker-0    | running | ::ffff:192.168.123.1 | 6233 |
+-------------+---------+----------------------+------+

```


# Running dev-scripts

The commands below include all steps required for deploying the environment with dev-scripts

```
(virtualenv) [root@titan47 linchpin-workspace]# ssh kni@provisionhost
Warning: Permanently added 'provisionhost,192.168.123.111' (ECDSA) to the list of known hosts.
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Wed Nov 13 22:40:34 2019 from 192.168.123.1
[kni@worker-1 ~]$
[kni@worker-1 ~]$ sudo subscription-manager register --serverurl=https://subscription.rhn.stage.redhat.com --username rhhi-next-qe --password redhat 
[kni@worker-1 ~]$ sudo subscription-manager attach --pool=8a99f9a96def8e1a016df3fd21a60519
[kni@worker-1 ~]$ sudo dnf install -y git make firewalld
[kni@worker-1 ~]$ sudo systemctl --now enable firewalld
[kni@worker-1 ~]$ git clone https://github.com/openshift-metal3/dev-scripts.git
[kni@worker-1 ~]$ cd dev-scripts/
[kni@worker-1 dev-scripts]$ cp config_example.sh config_$USER.sh
## Edit config file
[kni@worker-1 dev-scripts]$ vi config_$USER.sh 
#!/bin/bash

set +x
export PULL_SECRET='{ "auths": { "cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K21jb3JuZWFyZWRoYXRjb20xZmM0MHZzZ3h3cWVxa294dmpxeWk0eHF0ZXk6V0YxU0U2Rk82NlYzSE5CMVczUVFDOE9CSk81MFJGTFVNRVZPTUhUNEE2WVQ5MlVRRFJTTUZOVEQyNjYzRUQ3Vg==","email":"mcornea@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K21jb3JuZWFyZWRoYXRjb20xZmM0MHZzZ3h3cWVxa294dmpxeWk0eHF0ZXk6V0YxU0U2Rk82NlYzSE5CMVczUVFDOE9CSk81MFJGTFVNRVZPTUhUNEE2WVQ5MlVRRFJTTUZOVEQyNjYzRUQ3Vg==","email":"mcornea@redhat.com"},"registry.connect.redhat.com":{"auth":"NzQ4NDMyN3x1aGMtMUZDNDBWc0d4V3FFcWtPeFZKUXlJNFhRVGV5OmV5SmhiR2NpT2lKU1V6VXhNaUo5LmV5SnpkV0lpT2lJd01UWTRPRGhrTXpreE9UTTBOakpoWVdZNU1XUXdNalkwT0RCa01qWXhPU0o5LlBUVWlRcDRkMUozMW9maGpacm1zRjUxUDN5RGFZSzdOZC1FR0pxV0hHc0FIS0F0UlZ4ajZuUHl6LUlHb2RBdV8zUWdCX3hZRkN4NFVsZ3hheFhtVkNpU2k1NENvaVFyRTBkRlk5eHFxWUhwcXFTVlprS21LWGNPQ1NXRjF2OExRRTBJQVVEZm8xZlRtQ09KYUxsVnlVTnI3ZlFrYkdSRC1ZdWRhNEY2cFBZVHNlcnJnS1JsekRBRXNIdjJ4Rm9VcGJJYzVZb0czSzJuOVlBSFBhNHVVb25fckhDQTJadmZQb3V1ajdtZlZ2MjEtallXdGU0NTV5X1pZcjlfX095RElLSHdabjFqQ3Q5VG5IVC0yRFlQRl9WeThYek1GWTEwSmlqTmVHZEhmQkJ1RUtadFk5eS1SRjJ4QTlSNU9YcVNrdDczODRxWDd3VHBjWUh3MjlJeXVJREI2c0ZJaDFTWkdHTHpaZGJPUDBibzViMXV3YjRyTDhLRDlobld1SENkOVY0eC1RYjBQYlNYVDhRZEZSNVZCeG1BYkpGWHY5cU9lckRmckZWU0ZsVDByaVE0SmZKdVNUdUdYQWhwV25UREtyanJjYUpocmtsa09CSUx2YVNGVElCYl9QRkJxQmJaRnJmTkF0WHJZOU9DcmRZcEJWdVRwc01uaG84SWtyX09DSUhwRnFHdFZ4dkotRkkzeWtoOHEzWUt2aUNwd3VlVHhGY1JlOTdpMUxiYzlLV1JMWVhoNEVjVEtKenA0TjQ5WUM2aDRZM2JVZHpCeEU4bFVVQU55ZmlhWXFPa0pCUWRaOXNGb3VUa3JwU2xfUUFZN1l5LXpSNkxxZ3Z6QjF6UlFtemZZM2VtTmVYbHhXTjZtYUlpZ09NOFRmZm5ndlVldE45dl9tMDc4bmFv","email":"mcornea@redhat.com"},"registry.redhat.io":{"auth":"NzQ4NDMyN3x1aGMtMUZDNDBWc0d4V3FFcWtPeFZKUXlJNFhRVGV5OmV5SmhiR2NpT2lKU1V6VXhNaUo5LmV5SnpkV0lpT2lJd01UWTRPRGhrTXpreE9UTTBOakpoWVdZNU1XUXdNalkwT0RCa01qWXhPU0o5LlBUVWlRcDRkMUozMW9maGpacm1zRjUxUDN5RGFZSzdOZC1FR0pxV0hHc0FIS0F0UlZ4ajZuUHl6LUlHb2RBdV8zUWdCX3hZRkN4NFVsZ3hheFhtVkNpU2k1NENvaVFyRTBkRlk5eHFxWUhwcXFTVlprS21LWGNPQ1NXRjF2OExRRTBJQVVEZm8xZlRtQ09KYUxsVnlVTnI3ZlFrYkdSRC1ZdWRhNEY2cFBZVHNlcnJnS1JsekRBRXNIdjJ4Rm9VcGJJYzVZb0czSzJuOVlBSFBhNHVVb25fckhDQTJadmZQb3V1ajdtZlZ2MjEtallXdGU0NTV5X1pZcjlfX095RElLSHdabjFqQ3Q5VG5IVC0yRFlQRl9WeThYek1GWTEwSmlqTmVHZEhmQkJ1RUtadFk5eS1SRjJ4QTlSNU9YcVNrdDczODRxWDd3VHBjWUh3MjlJeXVJREI2c0ZJaDFTWkdHTHpaZGJPUDBibzViMXV3YjRyTDhLRDlobld1SENkOVY0eC1RYjBQYlNYVDhRZEZSNVZCeG1BYkpGWHY5cU9lckRmckZWU0ZsVDByaVE0SmZKdVNUdUdYQWhwV25UREtyanJjYUpocmtsa09CSUx2YVNGVElCYl9QRkJxQmJaRnJmTkF0WHJZOU9DcmRZcEJWdVRwc01uaG84SWtyX09DSUhwRnFHdFZ4dkotRkkzeWtoOHEzWUt2aUNwd3VlVHhGY1JlOTdpMUxiYzlLV1JMWVhoNEVjVEtKenA0TjQ5WUM2aDRZM2JVZHpCeEU4bFVVQU55ZmlhWXFPa0pCUWRaOXNGb3VUa3JwU2xfUUFZN1l5LXpSNkxxZ3Z6QjF6UlFtemZZM2VtTmVYbHhXTjZtYUlpZ09NOFRmZm5ndlVldE45dl9tMDc4bmFv","email":"mcornea@redhat.com"},"registry.svc.ci.openshift.org": { "auth": "c3lzdGVtLXNlcnZpY2VhY2NvdW50LWtuaS1kZWZhdWx0OmV5SmhiR2NpT2lKU1V6STFOaUlzSW10cFpDSTZJaUo5LmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyYm1raUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sWTNKbGRDNXVZVzFsSWpvaVpHVm1ZWFZzZEMxMGIydGxiaTAxZEdkbU55SXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMbTVoYldVaU9pSmtaV1poZFd4MElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVkV2xrSWpvaVlqZzNNRGt4WmpZdE5qRXlNeTB4TVdVNUxXRTJNVGt0TkRJd01UQmhPR1V3TURBeUlpd2ljM1ZpSWpvaWMzbHpkR1Z0T25ObGNuWnBZMlZoWTJOdmRXNTBPbXR1YVRwa1pXWmhkV3gwSW4wLm51VGR0RlczRENHcFpvT0pCbU45VjQwWG1wbmlZRE9tUnI2Z05vNGVwRVBrb1lDXzk1YmhWX0ttYjhoTnprOTNVTGtDNnJXNTVjTXFQMVM4RHh3QWw0RUxRZ2NFZXIyalBJLXZBNGUzdlZ5cHNLbS1XSkFxcWo2OGhNN0Z4ekMzRGgxY19lN19EQkJLOWtxZmcyRzZiNTJXQmI2RUhsODg2Q2Nza3JBVm1fbmprNS14ay1Ma1hSM3lXNW5JeXlZdXhNVGg1LUNMd3lQQy1yLVIzeklzdnlWelNPVTgyeUJaaE1tUmc3enUtOWlydThENHdqRFJQclhiSm1FV3lBM1FIUlJ2VTJuci01MTFEeEhEbWhtNW14YU0tSFA4emk3SU8zVEU5SU55S3BqTmo5eTIwNmtFN0NNSVNMWmRWWFl3MkpIQ1BmSmJQMHNJY3V0dnFvOTdGdw==" } } }'
set -x

NODES_PLATFORM=baremetal
INT_IF=eth1
PRO_IF=eth0
CLUSTER_PRO_IF=ens3
ROOT_DISK=/dev/sda
NODES_FILE=/home/kni/instackenv.json
MANAGE_BR_BRIDGE=n
NUM_WORKERS=1
CLUSTER_NAME=ostest
BASE_DOMAIN=test.metalkube.org
DNS_VIP=192.168.123.6
EXTERNAL_SUBNET=192.168.123.0/24

# Create json file to define nodes IPMI details
# Note: change MAC addresses according to the VMs created on your setup
# This file can also be copied from the hypervisor, in /tmp/ipmi_nodes.json

[kni@worker-1 dev-scripts]$ vi /home/kni/instackenv.json
{
  "nodes": [
    {
      "name": "openshift-master-0",
      "driver": "ipmi",
      "resource_class": "baremetal",
      "driver_info": {
        "username": "admin",
        "password": "password",
        "address": "ipmi://192.168.123.1:6230",
        "deploy_kernel": "http://172.22.0.1/images/ironic-python-agent.kernel",
        "deploy_ramdisk": "http://172.22.0.1/images/ironic-python-agent.initramfs"
      },
      "ports": [{
        "address": "52:54:00:31:bc:53",
        "pxe_enabled": true
      }],
      "properties": {
        "local_gb": "0",
        "cpu_arch": "x86_64"
      }
    }
        ,
          {
      "name": "openshift-master-1",
      "driver": "ipmi",
      "resource_class": "baremetal",
      "driver_info": {
        "username": "admin",
        "password": "password",
        "address": "ipmi://192.168.123.1:6231",
        "deploy_kernel": "http://172.22.0.1/images/ironic-python-agent.kernel",
        "deploy_ramdisk": "http://172.22.0.1/images/ironic-python-agent.initramfs"
      },
      "ports": [{
        "address": "52:54:00:0c:25:58",
        "pxe_enabled": true
      }],
      "properties": {
        "local_gb": "0",
        "cpu_arch": "x86_64"
      }
    }
        ,
          {
      "name": "openshift-master-2",
      "driver": "ipmi",
      "resource_class": "baremetal",
      "driver_info": {
        "username": "admin",
        "password": "password",
        "address": "ipmi://192.168.123.1:6232",
        "deploy_kernel": "http://172.22.0.1/images/ironic-python-agent.kernel",
        "deploy_ramdisk": "http://172.22.0.1/images/ironic-python-agent.initramfs"
      },
      "ports": [{
        "address": "52:54:00:06:c6:9a",
        "pxe_enabled": true
      }],
      "properties": {
        "local_gb": "0",
        "cpu_arch": "x86_64"
      }
    }
        ,
          {
      "name": "openshift-worker-0",
      "driver": "ipmi",
      "resource_class": "baremetal",
      "driver_info": {
        "username": "admin",
        "password": "password",
        "address": "ipmi://192.168.123.1:6233",
        "deploy_kernel": "http://172.22.0.1/images/ironic-python-agent.kernel",
        "deploy_ramdisk": "http://172.22.0.1/images/ironic-python-agent.initramfs"
      },
      "ports": [{
        "address": "52:54:00:1a:04:53",
        "pxe_enabled": true
      }],
      "properties": {
        "local_gb": "0",
        "cpu_arch": "x86_64"
      }
    }
        ]
}

## Run make

[kni@worker-1 dev-scripts]$ make

```

# Reaching the environment from the corporate network

In order to reach the environment from your laptop you need to set up NAT rules which forward traffic to the API and ingress VIPs. To set them up we can use the following playbook:

```shell
[root@sealusa2 ~]# cd ocp-edge-virt-env/
[root@sealusa2 ocp-edge-virt-env]# source virtualenv/bin/activate
(virtualenv) [root@sealusa2 ocp-edge-virt-env]# cd ocp-edge-demo/linchpin-workspace/
(virtualenv) [root@sealusa2 linchpin-workspace]# ansible-playbook -i inventories/ocp-edge.inventory hooks/ansible/ocp-edge-setup/iptables_console.yaml -e @extravars.yaml
```
