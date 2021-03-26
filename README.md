# NFS

How to create a PV/PVC read write many for a k8 POD. 


Create a quick linux NFS server
```
yum install nfs-util
chmod -R 755 /mnt/myexport
chown nfsnobody:nfsnobody /mnt/myexport

systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap

mkdir /mnt/myexport
vi /etc/exports with
	/mnt/myexport *(insecure,rw,no_root_squash,sync,no_subtree_check)
chmod -R 755 /mnt/myexport
chown nfsnobody:nfsnobody /mnt/myexport

systemctl restart nfs-server

showmount -e 192.168.3.4
Export list for 192.168.3.4:
/mnt/myexport *
```

Test the mount from another server (this is the same mount command the POD will do)
```
mkdir /ttmp/a
mount -t nfs -o hard,nfsvers=4.1 192.168.3.4:/mnt/myexport /tmp/a
```
To setup the CSI NFS Provisioner we followed the instructions on this page https://github.com/kubernetes-csi/csi-driver-nfs
```
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/install-driver.sh | bash -s master --
```
Check the status of the pods (in my case I had to scale to a 3 worker node k8 cluster)
```
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
```
Create the PV yaml file (note the last 2 lines have to change in your environment)
```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  csi:
    driver: nfs.csi.k8s.io
    readOnly: false
    volumeHandle: unique-volumeid  # make sure it's a unique id in the cluster
    volumeAttributes:
      server: 192.168.3.4
      share: /mnt/myexport
```













