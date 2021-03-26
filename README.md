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
