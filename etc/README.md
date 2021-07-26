See this link for formatting the drive and creating partitions
https://thesecmaster.com/how-to-partition-and-format-the-hard-drives-on-raspberry-pi/

Create partition:
- sudo parted
- print all
- select /dev/sda (this may be different in your case)
- mklabel gpt
- print 
- mkpart data-ext4 ext4 0% 100%
- q

Now format it.
- sudo mkfs.ext4 -L data-ext4  /dev/sda1

Now install the NFS server, I used this guide:
https://www.youtube.com/watch?v=EzqgJhu-qN8

Make a directory where you will mount your disk

- sudo mkdir /kubedata

Get the parition id
- sudo blkid /dev/sda1
/dev/sda1: LABEL="data-ext4" UUID="b80edf92-b9ab-44be-8c24-93d684c428db" TYPE="ext4" PARTLABEL="data-ext4" PARTUUID="e5b153e2-015d-4121-b30b-83d727347d47"
We need the PARTUUID for the next step

Mount the drive when the PI boots
- sudo nano /etc/fstab
add a line like this: PARTUUID=e5b153e2-015d-4121-b30b-83d727347d47 /kubedata ext4 defaults,noatime,nodiratime 0 2

Reboot or mount now
- sudo reboot
or
- sudo mount /kubedata

Install the NFS server 
- sudo apt install -y nfs-kernel-server

Make a directory to export
- sudo mkdir /kubedata/data

Export the directory to NFS
- sudo nano /etc/exports
Add this line:
/kubedata/data *(rw,sync,no_subtree_check)

Update exports
- sudo exportfs -ar

Now test the exports from each host in the K3s cluster

- sudo mount -t nfs 10.0.0.229:/kubedata/data /mnt

Test you can write to this directory, I opened up chmod 777 /kubedata/data on nfs server, will investigate later a better approach.

There is a cool project that will auto provision NFS storage for your K3s cluster: 
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

To install using Helm:
- helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
- helm repo update
- helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n nfs --create-namespace --set nfs.server=10.0.0.229 --set nfs.path=/kubedata/data

Note to update the nfs.server and nfs.path properties to yours.  Also this installs a bunch of stuff in the nfs namespace, as specificed in the helm install command.  Noteable the storage class 
- kubectl get sc 

NAME                 PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io                              Delete          Immediate           true                   8d
nfs-client           cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   112m

The nfs-client is the storage class we want to supply to any deployments in future that we want to use NFS.  
