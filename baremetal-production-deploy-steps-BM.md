# Introduction

This write-up will guide you through the process of deploying a BareMetal IPI installation of Red Hat OpenShift 4.x.

__Reference document:__ https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md

# Creating the test infrastructure

The following steps create the virtual test environment  infrastructure including VMs, libvirt networks and setting up the virtual BMCs:

* 6 virtual servers (1 provision node, 3 master and 2 worker nodes)
* DNS and DHCP servers will be run on hypervisor
* Each server will have 2 NICs pre-configured. NIC1 for the private network and NIC2 for the external network. NIC interface names need to be identical. See [issue](https://github.com/openshift/installer/issues/2762)
* Each server will have a RAID-1 configured and initialized
* Each server will have BMC configured __(IPMI)__
* Each server will have DHCP setup for external NICs
* Each server will have DNS setup for the API
* Each server will have PXE enabled

### Reserved IPs on DHCP Server
__cluster-name:__ ocp-edge

__domain:__ qe1.kni.lab.eng.bos.redhat.com

As an example given configuration for r640-u01.qe1.kni.lab.eng.bos.redhat.com setup
| Usage   |      Hostname      |  IP |
|----------|-------------|------|
| API | api.ocp-edge.qe1.kni.lab.eng.bos.redhat.com | 10.19.134.13 |
| Ingress LB (apps) |  *.apps.ocp-edge.qe1.kni.lab.eng.bos.redhat.com  | 10.19.??? |
| Nameserver(DNS) | ns1.ocp-edge.qe1.kni.lab.eng.bos.redhat.com | 10.19.134.14 |
| Provisioning node | provisioner.ocp-edge.qe1.kni.lab.eng.bos.redhat.com | 10.19.133.13 |
| Master-0 | ocp-edge-master-0.qe1.kni.lab.eng.bos.redhat.com | 10.19.133.14 |
| Master-1 | ocp-edge-master-1.qe1.kni.lab.eng.bos.redhat.com | 10.19.133.15 |
| Master-2 | ocp-edge-master-2.qe1.kni.lab.eng.bos.redhat.com | 10.19.133.16 |
| Worker-0 | ocp-edge-worker-0.qe1.kni.lab.eng.bos.redhat.com | 10.19.133.13 |

# Preparing the Provision node for OpenShift Install

__Note:__ Linchpin provision procedure creates "kni" user with sudo privileges.
          There is no any additional action required.
```bash
## Login into the provision node via ssh
ssh kni@provisioner
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
sudo dnf install -y patch libvirt qemu-kvm mkisofs python3-devel python3-pip gcc jq ipmitool firewalld bind-utils tar
## Modify the user to add the libvirt group to the kni user
sudo usermod --append --groups libvirt kni
## Enable and start firewalld, enable the http service, enable port 5000
sudo systemctl --now enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --add-port=5000/tcp --zone=libvirt  --permanent
sudo firewall-cmd --add-port=5000/tcp --zone=public   --permanent
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
PUB_CONN=$(nmcli con | awk '/eno2/ {print $1, $2}')
sudo nmcli con delete "$PUB_CONN" && sudo nmcli con add type bridge ifname baremetal autoconnect yes con-name baremetal stp off
PUB_CONN=$(nmcli con | awk '/eno2/ {print $1, $2, $3}')
sudo nmcli con delete "$PUB_CONN" && sudo nmcli con add type bridge-slave autoconnect yes con-name eno2 ifname eno2 master baremetal && sudo dhclient baremetal
PROV_CONN=$(nmcli con | awk '/{{ eno1 }}/ {print $1, $2, $3}')
sudo nmcli con delete "$PROV_CONN"
sudo nmcli con add type bridge ifname provisioning autoconnect yes con-name provisioning stp off
sudo nmcli con modify provisioning ipv4.addresses 172.22.0.1/24 ipv4.method manual
sudo nmcli con add type bridge-slave autoconnect yes con-name eno1 ifname eno1 master provisioning
sudo ifup provisioning     
sudo systemctl restart NetworkManager     
sudo systemctl restart libvirtd
## Verify the connection bridges have been properly created
sudo nmcli con show

NAME          UUID                                  TYPE      DEVICE       
baremetal     1ff8ae94-0584-43b1-8c1e-02d980ff3785  bridge    baremetal    
provisioning  3b1e3fb9-b8d5-44bd-8d9d-01626e36caf7  bridge    provisioning
virbr0        c05e9809-2c8a-4745-9558-bc8dbae7f3a7  bridge    virbr0       
eno1          7c4d4f36-336f-48bb-be2f-ea4d67314076  ethernet  eno1         
eno2          2f7f9ba8-daba-42f6-aebf-14c0694a7a7c  ethernet  eno2         
```
## Prepare your pull-secret
Download your pull-secret from try.openshift.com:

       Go to https://try.openshift.com/ -> Get Started -> Run on Bare Metal
       Click on “Installer-Provisioned Infrastructure”
       Click on “Copy Pull Secret”.  
       Create ~/pull-secret.json and paste it there
Add the api.ci.openshift.org auth to pull-secret:

       Obtain an API token by visiting https://api.ci.openshift.org/oauth/token/request -> Command Line Tools
       Copy the oc login cmd
       Paste in terminal that oc login cmd, then run this:
       ```
        [kni@worker-3 ~]$ oc registry login --to ~/pull-secret.json
       ```
       this will append the auth from  registry.svc.ci.openshift.org to your cloud.openshift.com pull-secret.


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
baseDomain: qe1.kni.lab.eng.bos.redhat.com
networking:
  machineCIDR: 10.19.134.0/24
metadata:
  name: ocp-edge-cluster
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 3
  platform:
    baremetal: {}
platform:
  baremetal:
    apiVIP: 10.19.134.13
    dnsVIP: 10.19.134.14
    ingressVIP: 10.19.134.15
    hosts:
      - name: openshift-master-0
        role: master
        bmc:
          address: ipmi://10.19.133.14
          username: root
          password: calvin
        bootMACAddress: 98:03:9b:61:7c:80
        hardwareProfile: default
      - name: openshift-master-1
        role: master
        bmc:
          address: ipmi://10.19.133.15
          username: root
          password: calvin
        bootMACAddress: 98:03:9b:61:7c:20
        hardwareProfile: default
      - name: openshift-master-2
        role: master
        bmc:
          address: ipmi://10.19.133.16
          username: root
          password: calvin
        bootMACAddress: 98:03:9b:61:85:e8
        hardwareProfile: default
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
```
export COMMIT_ID=$(./openshift-baremetal-install version | grep '^built from commit' | awk '{print $4}')
export RHCOS_PATH=$(curl -s -S https://raw.githubusercontent.com/openshift/installer/$COMMIT_ID/data/data/rhcos.json | jq .images.openstack.path | sed 's/"//g')
export RHCOS_URI=$(curl -s -S https://raw.githubusercontent.com/openshift/installer/$COMMIT_ID/data/data/rhcos.json | jq .baseURI | sed 's/"//g')
envsubst < metal3-config.yaml.sample > metal3-config.yaml
```
### 5. Create the OpenShift manifests
~~~
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
