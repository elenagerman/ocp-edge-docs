# Introduction

This write-up will guide you through the process of deploying a BareMetal IPI installation of Red Hat OpenShift 4.x.

__Reference document:__ https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md

# Creating the test infrastructure

The following steps create the virtual test environment  infrastructure including VMs, libvirt networks and setting up the virtual BMCs:

* 6 virtual servers (1 provision node, 3 master and 2 worker nodes)
* DNS and DHCP servers will be run on hypervisor
* Each server will have 2 NICs pre-configured. NIC1 for the private network and NIC2 for the external network. NIC interface names need to be identical. See [issue](https://github.com/openshift/installer/issues/2762)
* Each server will have BMC configured __(IPMI)__
* Each server will have DHCP setup for external NICs
* Each server will have DNS setup for the API

### Reserved IPs on DHCP Server
__cluster-name:__ ocp-edge-cluster

__domain:__ qe.lab.redhat.com

| Usage   |      Hostname      |  IP |
|----------|-------------|------|
| API | api.ocp-edge.qe.lab.redhat.com | 10.19.138.14 |
| Ingress LB (apps) |  *.apps.ocp-edge-cluster.qe.lab.redhat.com  | 10.19.138.15 |
| Nameserver(DNS) | ns1.ocp-edge-cluster.qe.lab.redhat.com | 10.19.138.16 |
| Provisioning node | provisioner.ocp-edge-cluster.qe.lab.redhat.com | 10.19.138.10 |
| Master-0 | ocp-edge-cluster-master-0.qe.lab.redhat.com | 10.19.138.11 |
| Master-1 | ocp-edge-cluster-master-1.qe.lab.redhat.com | 10.19.138.12 |
| Master-2 | ocp-edge-cluster-master-2.qe.lab.redhat.com | 10.19.138.13 |
| Worker-0 | ocp-edge-cluster-worker-0.qe.lab.redhat.com | 10.19.138.8 |
| Worker-1 | ocp-edge-cluster-worker-1.qe.lab.redhat.com | 10.19.138.9 |

```bash
mkdir ocp-edge-virt-env
cd ocp-edge-virt-env/
yum install -y libvirt-devel python3-virtualenv gcc git python3-libvirt
dnf localinstall -y http://download.eng.bos.redhat.com/brewroot/vol/rhel-8/packages/sshpass/1.06/3.el8ae/x86_64/sshpass-1.06-3.el8ae.x86_64.rpm
virtualenv virtualenv
source virtualenv/bin/activate
pip install ansible==2.8 linchpin==1.7.6.2 libvirt-python netaddr lxml
curl -o virtualenv/bin/install_selinux_venv.sh https://raw.githubusercontent.com/CentOS-PaaS-SIG/linchpin/1792bb8fa02c4acbef63c987f715f0c3cf8b193e/scripts/install_selinux_venv.sh; bash -x virtualenv/bin/install_selinux_venv.sh
git -c http.sslVerify=false clone https://gitlab.cee.redhat.com/ocp-edge-qe/ocp-edge-demo.git
cd ocp-edge-demo/linchpin-workspace/
```
__IMPORTANT:__
   Edit custom vars in hooks/ansible/ocp-edge-setup/extravars.yaml 
   Take care about cluster_name, base_domain and cluster_domain are set correctly
   For example: cluster_name = ocp-edge-cluster; base_domain = qe.lab.redhat.com
                cluster_domain = ocp-edge-cluster.qe.lab.redhat.com
```
## Edit custom vars in hooks/ansible/ocp-edge-setup/extravars.yaml if needed
ansible -c local localhost -m template -a "src=hooks/ansible/ocp-edge-setup/extravars.yaml dest=$PWD/extravars.yaml" -e @hooks/ansible/ocp-edge-setup/extravars.yaml
linchpin --template-data @extravars.yaml -v destroy libvirt-network libvirt-new
linchpin --template-data @extravars.yaml -v up libvirt-network libvirt-new cfgs
virsh list --all

 Id    Name                           State
----------------------------------------------------
 15    worker-3                       running
 -     master-0                       shut off
 -     master-1                       shut off
 -     master-2                       shut off
 -     worker-0                       shut off
 -     worker-1                       shut off
 -     worker-2                       shut off

virsh net-list

  Name                 State      Autostart     Persistent
 ----------------------------------------------------------
  baremetal            active     no            yes
  default              active     yes           yes
  provisioning         active     no            yes

/root/.virtualenvs/vbmc/bin/python /root/.virtualenvs/vbmc/bin/vbmc list
  +-------------+---------+----------------------+------+
  | Domain name | Status  | Address              | Port |
  +-------------+---------+----------------------+------+
  | master-0    | running | ::ffff:192.168.123.1 | 6230 |
  | master-1    | running | ::ffff:192.168.123.1 | 6231 |
  | master-2    | running | ::ffff:192.168.123.1 | 6232 |
  | worker-0    | running | ::ffff:192.168.123.1 | 6233 |
  | worker-1    | running | ::ffff:192.168.123.1 | 6234 |
  | worker-2    | running | ::ffff:192.168.123.1 | 6235 |
  +-------------+---------+----------------------+------+
```

# Preparing the Provision node for OpenShift Install

__Note:__ Linchpin provision procedure creates "kni" user with sudo privileges.
          There is no any additional action required.
```bash
## Login into the provision node via ssh
ssh kni@provisionhost
## Create an ssh key for the new user
sudo ssh-keygen -t rsa -f /home/kni/.ssh/id_rsa -N ''
## Register your environment using subscription manager
sudo subscription-manager register --serverurl=https://subscription.rhn.stage.redhat.com --username rhhi-next-qe --password redhat
sudo subscription-manager attach --pool=8a99f9a96def8e1a016df3fd21a60519
sudo subscription-manager refresh
## Create BaseOS hardwareProfile
echo -e "[rhelosp-rhel-8.1-BaseOS]\nname=Red Hat Enterprise Linux $releasever - \$basearch - ServerMirror\nbaseurl=http://download.engineering.redhat.com/pub/rhel/released/RHEL-\$releasever/\$releasever.1.0/BaseOS/\$basearch/os/\nenabled=1\ngpgcheck=0\npriority=30" | sudo dd of=/etc/yum.repos.d/mirror-baseos.repo
## Create AppStream repo
echo -e "[rhelosp-rhel-8.1-App]\nname=Red Hat Enterprise Linux \$releasever - \$basearch - ServerMirror\nbaseurl=http://download.engineering.redhat.com/pub/rhel/released/RHEL-\$releasever/\$releasever.1.0/AppStream/\$basearch/os/\nenabled=1\ngpgcheck=0\npriority=30" | sudo dd of=/etc/yum.repos.d/mirror-appstream.repo
```
```bash
## Install the following packages
sudo dnf install -y patch libvirt qemu-kvm mkisofs python3-devel python3-pip gcc jq ipmitool firewalld bind-utils git
## Modify the user to add the libvirt group to the kni user
sudo usermod --append --groups libvirt kni
## Enable and start firewalld, enable the http service, enable port 5000
sudo systemctl --now enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --add-port=5000/tcp --zone=libvirt --permanent
sudo firewall-cmd --add-port=5000/tcp --zone=public --permanent
sudo firewall-cmd --add-port=6230-6238/udp --zone=libvirt --permanent
sudo firewall-cmd --reload
## Start and enable the libvirtd service
sudo systemctl start libvirtd
sudo systemctl enable libvirtd --now
## Create the default storage pool and start it
sudo virsh pool-define-as --name default --type dir --target /var/lib/libvirt/images
sudo virsh pool-start default
sudo virsh pool-autostart default
## Configure networking
PUB_CONN=$(nmcli con | awk '/eth1/ {print $1, $2}')
sudo nmcli con delete "$PUB_CONN" && sudo nmcli con add type bridge ifname baremetal autoconnect yes con-name baremetal stp off
PUB_CONN=$(nmcli con | awk '/eth1/ {print $1, $2, $3}')
sudo nmcli con delete "$PUB_CONN" && sudo nmcli con add type bridge-slave autoconnect yes con-name eth1 ifname eth1 master baremetal && sudo dhclient baremetal
#sudo nmcli con down baremetal && pkill dhclient && dhclient baremetal && sudo ifup baremetal
PROV_CONN=$(nmcli con | awk '/eth0/ {print $1, $2, $3}')
sudo nmcli con delete "$PROV_CONN"
sudo nmcli con add type bridge ifname provisioning autoconnect yes con-name provisioning stp off
sudo nmcli con modify provisioning ipv4.addresses 172.22.0.1/24 ipv4.method manual
sudo nmcli con add type bridge-slave autoconnect yes con-name eth0 ifname eth0 master provisioning
sudo ifup provisioning     
sudo systemctl restart NetworkManager     
sudo systemctl restart libvirtd
## Verify the connection bridges have been properly created
sudo nmcli con show

NAME          UUID                                  TYPE      DEVICE       
baremetal     ec00b398-d609-4e9c-82c5-bf36567e856e  bridge    baremetal    
provisioning  0cbb4291-e7ad-4423-a0ef-73387b3458aa  bridge    provisioning
virbr0        8aa0f225-f7a3-4646-a1c3-d94979163608  bridge    virbr0       
eth0          41fbc932-d84f-43e8-9f1a-cce738063085  ethernet  eth0         
eth1          c53ecf11-d445-41ad-bb94-69c062f7faac  ethernet  eth1                   
```
## Prepare your pull-secret
Download your pull-secret from try.openshift.com:

       Go to https://try.openshift.com/ -> Get Started -> Run on Bare Metal
       Click on “Installer-Provisioned Infrastructure”
       Click on “Copy Pull Secret”.  
       Create ~/pull-secret.json and paste it there
Add the api.ci.openshift.org auth to pull-secret:
       Edit ~/pull-secret.json and add api.ci.openshift.org auth
```bash
"registry.svc.ci.openshift.org":{"auth": "c3lzdGVtLXNlcnZpY2VhY2NvdW50LWlwdjYtZGVmYXVsdDpleUpoYkdjaU9pSlNVekkxTmlJc0ltdHBaQ0k2SWlKOS5leUpwYzNNaU9pSnJkV0psY201bGRHVnpMM05sY25acFkyVmhZMk52ZFc1MElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl1WVcxbGMzQmhZMlVpT2lKcGNIWTJJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJsWm1GMWJIUXRkRzlyWlc0dGJEbHhhM0VpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdWbVlYVnNkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJakk0WWpZMk0yUXlMV1pqT1dZdE1URmxPUzA1T0dFNUxUUXlNREV3WVRobE1EQXdNeUlzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwcGNIWTJPbVJsWm1GMWJIUWlmUS5JTG0tSWQwUUU3cGRsbHBZWmJVWmdoUWptYzJjY19BRWc3WEdlUVhNZ2U5VzV0NmNOVi1uUzc5UXNyVF85ODRqWHlzWThhcFNYR1hFTjY4TC1VeXNmQWNHNEpUSnJ6aks4Mm9uN0d1MjEtWXRRYldJT3hwMnNJb2w4QU5QcWxrNTlncFljNEJYMVhDMVR5WFdBQmpqSWVIQ2QtWWU5ZEYtZ3FySVlrbWZySHZ5SlBzMEVJc1BSXzN4bElFU0FnaDFfMFFSbzk5dkc0Y2xBcFk5R0YxUS1oNzNucnNkX1VpQVFDUFA2Q0FjcDhFXzlKLVdNaVBPbFRUOWJQaEg2emRMa2hzR0E0NXNVZC0tcW9aV1RZRXdSeFgwd21iWkQ0YlZSbDdOMXlmRGxqZm8xcWtMZlBzZXR0MDFuaFo5NjBHZzFCVW9NeVJXcW52cElLcGRBM2FwUUE="}
```

# Retrieving the OpenShift Installer

Two approaches:
    1. Choose a successfully deployed release that passed CI
    2. Deploy latest

## Choosing a OpenShift Installer Release from CI
1. Go to https://openshift-release.svc.ci.openshift.org/ and choose a release which has passed the tests for metal.
2. Save the release name. e.g: 4.3.0-0.nightly-2019-12-29-173422
3. Configure VARS
```
export VERSION=4.3.0-0.nightly-2019-12-29-173422
export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp-dev-preview/$VERSION/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}' | xargs)
```
## Choosing the latest OpenShift installer
1. Configure VARS
```
export VERSION=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp-dev-preview/latest/release.txt | grep 'Name:' | awk -F: '{print $2}' | xargs)
export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp-dev-preview/latest/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}' | xargs)
```
## Extract the Installer
```
export cmd=openshift-baremetal-install
export pullsecret_file=~/pull-secret.json
export extract_dir=$(pwd)
# Get the oc binary
curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp-dev-preview/$VERSION/openshift-client-linux-$VERSION.tar.gz | tar zxvf - oc
sudo cp ./oc /usr/local/bin/oc
# Extract the baremetal installer
oc adm release extract --registry-config "${pullsecret_file}" --command=$cmd --to "${extract_dir}" ${RELEASE_IMAGE}
```

## Configure the install-config and metal3-configured
### 1. Configure the install-config.yaml (Make sure you change the pullSecret and sshKey)
~~~yaml
   apiVersion: v1
   baseDomain: <domain>
   metadata:
     name: <cluster-name>
   networking:
     machineCIDR: <public-cidr>
   compute:
   - name: worker
     replicas: 2
   controlPlane:
     name: master
     replicas: 3
     platform:
       baremetal: {}
   platform:
     baremetal:
       apiVIP: <api-ip>
       ingressVIP: <wildcard-ip>
       dnsVIP: <dns-ip>
       provisioningBridge: provisioning
       externalBridge: baremetal
       hosts:
         - name: openshift-master-0
           role: master
           bmc:
             address: ipmi://<out-of-band-ip>
             username: <user>
             password: <password>
           bootMACAddress: <NIC1-mac-address>
           hardwareProfile: default
         - name: openshift-master-1
           role: master
           bmc:
             address: ipmi://<out-of-band-ip>
             username: <user>
             password: <password>
           bootMACAddress: <NIC1-mac-address>
           hardwareProfile: default
         - name: openshift-master-2
           role: master
           bmc:
             address: ipmi://<out-of-band-ip>
             username: <user>
             password: <password>
           bootMACAddress: <NIC1-mac-address
           hardwareProfile: default
         - name: openshift-worker-0
           role: worker
           bmc:
             address: ipmi://<out-of-band-ip>
             username: <user>
             password: <password>
           bootMACAddress: <NIC1-mac-address
           hardwareProfile: unknown
         - name: openshift-worker-1
           role: worker
           bmc:
             address: ipmi://<out-of-band-ip>
             username: <user>
             password: <password>
           bootMACAddress: <NIC1-mac-address
           hardwareProfile: unknown
   pullSecret: '<pull_secret>'
   sshKey: '<ssh_pub_key>'
   ~~~
__Example:__

~~~yaml
apiVersion: v1
baseDomain: qe.lab.redhat.com
networking:
  machineCIDR: 192.168.123.0/24
metadata:
  name: ocp-edge-cluster
compute:
- name: worker
  replicas: 3
controlPlane:
  name: master
  replicas: 3
  platform:
    baremetal: {}
platform:
  baremetal:
    apiVIP: 192.168.123.5
    dnsVIP: 192.168.123.6
    ingressVIP: 192.168.123.10
    provisioningBridge: provisioning
    externalBridge: baremetal
    hosts:
      - name: openshift-master-0
        role: master
        bmc:
          address: ipmi://192.168.123.1:6230
          username: admin
          password: password
        bootMACAddress: 52:54:00:2d:cb:b2
        hardwareProfile: default
      - name: openshift-master-1
        role: master
        bmc:
          address: ipmi://192.168.123.1:6231
          username: admin
          password: password
        bootMACAddress: 52:54:00:81:dc:35
        hardwareProfile: default
      - name: openshift-master-2
        role: master
        bmc:
          address: ipmi://192.168.123.1:6232
          username: admin
          password: password
        bootMACAddress: 52:54:00:0d:4a:8e
        hardwareProfile: default
      - name: openshift-worker-0
        role: worker
        bmc:
          address: ipmi://192.168.123.1:6233
          username: admin
          password: password
        bootMACAddress: 52:54:00:56:c9:8e
        hardwareProfile: unknown
      - name: openshift-worker-1
        role: worker
        bmc:
          address: ipmi://192.168.123.1:6234
          username: admin
          password: password
        bootMACAddress: 52:54:00:92:3a:88
        hardwareProfile: unknown
      - name: openshift-worker-2
        role: worker
        bmc:
          address: ipmi://192.168.123.1:6235
          username: admin
          password: password
        bootMACAddress: 52:54:00:3b:e1:01
        hardwareProfile: unknown
pullSecret: ''
sshKey: ''
~~~
__Note:__ MAC addresses could be taken from /tmp/ipmi_nodes.json

### 2. Create a directory to store cluster configs
```
mkdir ~/ocp
cp install-config.yaml ~/ocp
```

### 3. IMPORTANT: This portion is critical as the OpenShift installation won't complete without the metal3-operator being fully operational. This is due to this [issue](https://github.com/openshift/installer/pull/2449) we need to fix the ConfigMap for the Metal3 operator. This ConfigMap is used to notify `ironic` how to PXE boot new nodes.

Create the sample ConfigMap `metal3-config.yaml.sample`
~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metal3-config
  namespace: openshift-machine-api
data:
  cache_url: ''
  deploy_kernel_url: http://172.22.0.3:6180/images/ironic-python-agent.kernel
  deploy_ramdisk_url: http://172.22.0.3:6180/images/ironic-python-agent.initramfs
  dhcp_range: 172.22.0.10,172.22.0.100
  http_port: "6180"
  ironic_endpoint: http://172.22.0.3:6385/v1/
  ironic_inspector_endpoint: http://172.22.0.3:5050/v1/
  provisioning_interface: <NIC1>
  provisioning_ip: 172.22.0.3/24
  rhcos_image_url: ${RHCOS_URI}${RHCOS_PATH}
~~~

__NOTE:__ The `provision_ip` should be modified to an available IP on the `provision` network. The default is `172.22.0.3`

### 4. Create the final ConfigMap
```sh
export COMMIT_ID=$(./openshift-baremetal-install version | grep '^built from commit' | awk '{print $4}')
export RHCOS_PATH=$(curl -s -S https://raw.githubusercontent.com/openshift/installer/$COMMIT_ID/data/data/rhcos.json | jq .images.openstack.path | sed 's/"//g')
export RHCOS_URI=$(curl -s -S https://raw.githubusercontent.com/openshift/installer/$COMMIT_ID/data/data/rhcos.json | jq .baseURI | sed 's/"//g')
envsubst < metal3-config.yaml.sample > metal3-config.yaml
```
### 5. Create the OpenShift manifests
~~~sh
./openshift-baremetal-install --dir ~/ocp create manifests
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
WARNING Discarding the Openshift Manifests that was provided in the target directory because its dependencies are dirty and it needs to be regenerated
~~~
### 6. Copy the `metal3-config.yaml` to the `ocp/openshift` directory
~~~sh
cp ~/metal3-config.yaml ocp/openshift/99_metal3-config.yaml
~~~
## Deploying Routers on Worker Nodes
__NOTE:__ By default two routers are deployed. If you have two worker nodes already, this section may be skipped. For more info on ingress operator visit: https://docs.openshift.com/container-platform/4.2/networking/ingress-operator.html

The router-replicas.yaml file
```
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  replicas: <num-of-router-pods>
  endpointPublishingStrategy:
    type: HostNetwork
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/worker: ""
```
__NOTE__: If working with just one worker node, set this value to one. If working with more than 3+ workers, additional router pods (default 2) may be recommended.

Once the router-replicas.yaml file has been saved, copy the file to the clusterconfigs/openshift directory.
```
cp ~/router-replicas.yaml ocp/openshift/99_router-replicas.yaml
```

## Deploying the Cluster via the OpenShift Installer
Run the OpenShift Installer
```
./openshift-baremetal-install --dir ~/ocp --log-level debug create cluster
```
