#Enabling the internal registry

In OCP4.3 and up, the installer detects if the environment is baremetal - and in such a case disables the registry. This will happen in all our libvirt environments as well. You will see errors such as this:

bc/ruby-hello-world is pushing to istag/ruby-hello-world:latest, but the administrator has not configured the integrated container image registry.

Here are the steps I took to get my registry working:
1) Created an NFS share on the host:
mkdir /mnt/export_dir
chown nfsnobody:nfsnobody /mnt/export_dir
chmod 700 /mnt/export_dir

2) In /etc/exports.d/export_dir.exports:
/mnt/export_dir *(rw,async,all_squash,no_wdelay,insecure,fsid=0)

3) Run th exportfs command on the host, and if needed also start the nfs service:
exportfs -a
systemctl enable --now nfs

4) Create a PV using NFS, with 100GiB at least:
apiVersion: v1
kind: PersistentVolume
metadata:
  name: titan90data
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /mnt/export_dir
    server: titan90.lab.eng.tlv2.redhat.com
  persistentVolumeReclaimPolicy: Recycle

5) Change the registry configuration, and set managementState and the storage as in the below. Leave the claim field blank so that a PVC will be created automatically:
oc edit configs.imageregistry.operator.openshift.io
spec:
  managementState: Managed
  storage:
    pvc:
      claim:

6) Look at the ConfigMaps in the openshift-image-registry namespace, and add the certificate from the "serviceca" configmap to the list of certificates in the "trusted-ca" configmap
