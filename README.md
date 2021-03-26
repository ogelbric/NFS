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
```
[root@orfdns ~]# kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
NAME                                  READY   STATUS    RESTARTS   AGE   IP             NODE                                           NOMINATED NODE   READINESS GATES
csi-nfs-controller-6986f6fb4c-2h4m9   3/3     Running   0          16h   192.168.7.54   tkg-newberlin-workers-rxbc4-795fcf98cc-dzjpl   <none>           <none>
csi-nfs-controller-6986f6fb4c-gpsbp   3/3     Running   0          16h   192.168.7.56   tkg-newberlin-workers-rxbc4-795fcf98cc-npwqf   <none>           <none>
[root@orfdns ~]# kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
NAME                 READY   STATUS    RESTARTS   AGE   IP             NODE                                           NOMINATED NODE   READINESS GATES
csi-nfs-node-gjq2z   3/3     Running   0          16h   192.168.7.53   tkg-newberlin-workers-rxbc4-795fcf98cc-lld9n   <none>           <none>
csi-nfs-node-hctkq   3/3     Running   0          16h   192.168.7.54   tkg-newberlin-workers-rxbc4-795fcf98cc-dzjpl   <none>           <none>
csi-nfs-node-ng9ht   3/3     Running   0          16h   192.168.7.56   tkg-newberlin-workers-rxbc4-795fcf98cc-npwqf   <none>           <none>
[root@orfdns ~]# 
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
Create the PVC
```
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-nfs-static
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-nfs
  storageClassName: ""
```
Sample POD (note the image line could look like this: image: nginx)
```
apiVersion: v1
kind: Pod
metadata:
  name: task1-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: pvc-nfs-static
  containers:
    - name: task-pv-container
      image: gcr.io/unreal-snow-xxxx/nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
Apply the 3 yaml files
```
kubectl apply -f ./nfspv.yaml
kubectl apply -f ./nfspvc.yaml
kubectl apply -f ./nfspod.yaml
 
```
Check out the PV/PVC
```
[root@orfdns ~]# kubectl get pv,pvc | grep nfs
persistentvolume/pv-nfs   10Gi       RWX            Retain           Bound    default/pvc-nfs-static                           37m
persistentvolumeclaim/pvc-nfs-static   Bound    pv-nfs   10Gi       RWX                           36m
[root@orfdns ~]# 
```
Check out if pod is running
```
[root@orfdns ~]# kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
task1-pv-pod   1/1     Running   0          39m
[root@orfdns ~]# 
```
Jump into the POD to test (note the mount point is there)
```
kubectl exec --stdin --tty task1-pv-pod -- /bin/bash


root@task1-pv-pod:/# df -h
Filesystem                 Size  Used Avail Use% Mounted on
overlay                     16G  4.5G   11G  31% /
tmpfs                       64M     0   64M   0% /dev
tmpfs                      2.0G     0  2.0G   0% /sys/fs/cgroup
shm                         64M     0   64M   0% /dev/shm
/dev/root                   16G  4.5G   11G  31% /etc/hosts
192.168.3.4:/mnt/myexport   14G  5.6G  7.8G  42% /usr/share/nginx/html
tmpfs                      2.0G   12K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                      2.0G     0  2.0G   0% /proc/acpi
tmpfs                      2.0G     0  2.0G   0% /sys/firmware
```
Write something to the directory / NFS file system
```
root@task1-pv-pod:/# cd /usr/share/nginx/html
```
```
root@task1-pv-pod:/usr/share/nginx/html# ls
abc  xyz
```
```
root@task1-pv-pod:/usr/share/nginx/html# date >> abc
root@task1-pv-pod:/usr/share/nginx/html# cat abc
Fri Mar 26 12:26:52 UTC 2021
Fri Mar 26 12:27:01 UTC 2021
Fri Mar 26 12:27:12 UTC 2021
Fri Mar 26 12:58:20 UTC 2021
root@task1-pv-pod:/usr/share/nginx/html# 
```









