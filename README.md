Why and I doing this?  I work in tech and clustering interests me, also if my entire network is going to depend on a rasperberry pi for DNS, ad blocking, and DHCP, I might as well have some redundacy.
Is this a lot of overhead for redundancy, maybe, but again, it interests me, so why not :-) 
Also we all work from home now, we need the network to stay up and be reliable.

This is not a step by step guide but if you have questions feel free to add an issue and I will try and help.  I do presume you have some knowledge of Kubernetes and helm. 
I had no hands on practice when I started this but YouTube and other sources like (https://github.com/geerlingguy), (https://github.com/justmeandopensource) and (https://github.com/timothystewart6) taught me a lot, so thanks to them.  Check out their YouTube Channels too.

My OS of choice is RaspberryPI OS beta 64bit, seems stable and light weight enough for use.  
I decided to use K3s (https://k3s.io) for the Kubernetes runtime.  It's lightweight and seems to be stable.  Easy to setup and install.

Hardware is 3 8GB PIs with 64GB EVO PLUS SD cards.  3 POE PI Hats and 1 POE switch.  The nodes are configured as 1 K3s Master and 2 K3s workers nodes (Muppet themed lol).  The master will be a worker node as well.  
TODO: add redundacy to master node.  I plan to do this overtime, perfer 2 more master nodes with embedded database for high availability now that etcd is supported in K3s.

For each PI, I flashed the SD with 2021-05-07-raspios-buster-arm64-lite.zip using the Raspnery PI imager tool, I endabled SSH access by dropping in a ssh file into the SD card.

As well I configured each pi:
- Set hostname - using sudo raspi-config
- set password - using sudo raspi-config
- Set Timezone - - using sudo raspi-config under localisation
- Set static IP address - sudo nano /etc/dhcpcd.conf
  interface eth0
  static ip_address=10.0.0.226/24
  static routers=10.0.0.1
  static domain_name_servers=10.0.0.1 1.1.1.1 1.0.0.1
- sudo reboot
Copy SSH Key over - using ssh-copy-id [remote_username]@[server_ip_address] (assuming you already generated a key on your client), if not, see this link, https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md 
- Update the PI
  sudo apt update
  sudo apt full-upgrade
- disable swap as Kubernetes has no support for it. sudo systemctl disable dphys-swapfile.service
- sudo reboot

Hosts:
kermit - 10.0.0.226
fozzie - 10.0.0.227
gonzo - 10.0.0.228

Raspberry OS defaults to using nftables instead of iptables. K3S networking features require iptables and do not work with nftables. Follow the steps below to switch configure the OS to use legacy iptables:

- sudo iptables -F
- sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
- sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
- sudo reboot

Standard Raspberry OS installations do not start with cgroups enabled. K3S needs cgroups to start the systemd service. cgroups can be enabled by appending cgroup_memory=1 cgroup_enable=memory to /boot/cmdline.txt.

- sudo nano /boot/cmdline.txt, append cgroup_memory=1 cgroup_enable=memory to the end of the line like so:
console=serial0,115200 console=tty1 root=PARTUUID=58b06195-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory
- sudo reboot

I prefer to use MetalLB instead of the stock K3s load balancer (Klipper) becuase MetalLB will assign 1 IP address per service for the entire cluster.  
I do not need local storage, as it defeats the pupose of failover if the data for the pod isn't also on the node the pod is running on, I will be using another storage solution later in this document.  
I also prefer to use nginx ingress instead of Traefik.  Both of these swaps are optional.  It's just preference.  If you swap them, you need to install K3s without these options:

I installed master node with 
- curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC=" --disable traefik --disable servicelb --disable local-storage" sh -

I installed worker nodes with 
- curl -sfL https://get.k3s.io | K3S_URL=https://<myserver>:6443 K3S_TOKEN=<mynodetoken> sh -
You can checkout the quick start guide on the K3s website.  Be sure to replace the values above with your own.  

Be sure to configure your client machine for kubectl and helm so you can access your cluster, you will need them.  If you are on mac, you can use homebrew to install both.

Now you are ready to install MetalLB as your bare metal load balancer, you will need a block of IP addresses your current DHCP server will not assign to other devices on your network, once you have it you are ready to install MetalLB.
I guess I wasn't too comfotable with Helm yet so I used the manifests on MetalLB's website, https://metallb.org/installation/ 

- helm repo add metallb https://metallb.github.io/metallb
- helm repo update
- helm show values metallb/metallb >> values.yaml
- I edited the values.yaml to add my IP range, per doc
  configInline:
  address-pools:
   - name: default
     protocol: layer2
     addresses:
     - 10.0.0.2-10.0.0.11
- helm install metallb metallb/metallb -f values.yaml

This is the simplest way to get metallb up and running with a block of addresses for it to hand out to loadbalancer services.  Now metallb is ready and you can test it by deploying an nginx pod and loadbalancer service.
You can further test by deploying 2 nginx pods and 2 LB services to ensure metallb is giving out the IPs correctly. (notes that you can share IPs for a deployment that has more than 1 service like pihole (DNS, Web, DHCP), so 10 addresses seemed like a good number, but we will see)
e.g.
- kubectl create deployment nginx --image=nginx
- kubectl expose deploy nginx --port 80 --type LoadBalancer

Do a second one :-)
- kubectl create deployment nginx2 --image=nginx
- kubectl expose deploy nginx2 --port 80 --type LoadBalancer

Check to see if the LB is working
- kubectl get all
- curl 10.0.0.2
- curl 10.0.0.3

Delete the deployment and serviecs
- kubectl delete deploy nginx nginx2
- kubectl svc deploy nginx nginx2

A good cluster needs some monitoring.  I have choosen to use https://github.com/carlosedp/cluster-monitoring by https://github.com/carlosedp
This is a tuned monitoring solution for Raspberry Pis, and K3s.  You can use the monitoring manifests in this repo.  
You will need build-essential installed on your host where you are compiling this project.  
- sudo apt update && sudo apt install build-essentail -y
You will need to edit the vars.jsonet file per the quick start guide on the link above.  then run:
- sudo make vendor
- sudo make
- kubectl apply -f manifests/setup/
wait a minute for these resources to come alive, you can check kubectl get all -monitoring
- kubectl apply -f manifests/

Note, I could not get ingress to work and I am not sure if this is because I don't use Traefik by default and instead ingress-nginx or soem other reason.  So I deleted the ingress resource YAML files from manifests directory.  I also edited the grafana-service.yaml and changed the service type to LoadBalancer, since i have MetalLB, this was a no-brainer.  
You can access Grafana on http://<loadbalancerIP>:3000  default username and password are admin/admin
You can find the loadblanacerIP by issuing 
- kubectl get svc -n monitoring

At this point, I needed a storage solution, I don't own a NAS, so NFS was out of the question, so I went with Longhorn.  Longhorn is a  block storage system that can replicate data around to all the worker nodes in your K3s cluster.  Pretty simple install using helm.  
There are some prerequisits to using longhorn (I had all of them already excpet for iscsi), all hosts that will keep storage replicas (for me this is all 3) will need have open-iscsi installed, it's easy to install, 

sudo apt install open-iscsi.  

As well longhorn requires an ingress controller to access the frontend UI, I deployed ingress-nginx.

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
kubectl get all -n ingress-nginx

Now you can install longhorn:
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn -n longhorn-system --create-namespace
kubectl get all -n longhorn-system

Setup ingress for longhorn

Create a basic auth file:
- USER=<USERNAME_HERE> && PASSWORD=<PASSWORD_HERE> && echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
Create a secret:
- kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
Create the ingress:
- kubectl -n longhorn-system apply -f longhorn-ingress.yml
Yaml is in this repo under longhorn.
test the ingress:
- Kubectl get svc ingress-nginx-controller -n ingress-nginx
- curl <external-ip>

Now we are ready to deploy for the services: I have chosen to start with 3 services
1. Pihole (https://github.com/MoJo2600/pihole-kubernetes) - I used this helm chart to deploy pihole, very nice and credit to https://github.com/MoJo2600 and all the contributors for all their hard work.  You can access my pihole values yaml file in this repo
- helm install --namespace pihole --values pihole.yaml pihole mojo2600/pihole --create-namespace
test pihole
- Kubectl get svc pihole-web -n pihole
- curl <external-ip>
You can also test further by setting your local computer's DNS to the external-ip address. Then go to website you don't often visit.


2. Unbound, I had some work to do here, no helm chart exists that I could find, no kubernetes manifests either.  Just some 4 year old stuff I saw on github.  So I wrote my own manifests based on the great work that https://github.com/MatthewVance has done to provide unbound on arm (https://github.com/MatthewVance/unbound-docker-rpi).  

You can access my unbound manfest files in this repo.  
To apply the manifests run these two commands
- kubectl apply -f unbound.yaml -n pihole
- kubectl apply -f unbound-svc.yaml -n pihole
To check it
- kubectl get all -n pihole

Yes I know this can all go into one file and namespaces can be hardcoded, I'm lazy :-).  
Further, since the Unbound container can move around the cluster, the service defintion has the ClusterIP hard coded, it's an internal IP to the cluster.  Make sure no other service has this IP in your cluster.  I wanted to have a "Statically" configured service IP so that pihole could always find the Unbound service

3. Home Assistant (https://k8s-at-home.com), credit to this project and all teh 150+ contributors for creating an easy to use helm charts like HA.  You can see my values file in this repo.
To install HA:
helm install home-assistant -n home-assistant k8s-at-home/home-assistant --values values.yaml --create-namespace

Failover testing.  I was able to drain a node (kubectl drain kermit --ignore-daemonsets --delete-local-data) that services were running on and watch them failover to another node, there is a short interuption to my network and internet access but this is much better than scrambling to get a new pi online or killing the pihole alltogether and removing it from my setup, even if temporarily.  Also I run a lot of lights off of Home assistant, will be nice not to set that up again in case of hardware failure (pi board, sd card, etc).  Failovers work, longhorn replication works, it's all working good... until it's not :-)

Update July 21.

I added another Raspberry PI 4 2GB to the deployment, it's acting as an NFS server:
dr-bunsen-honeydew - 10.0.0.229 (non-POE-hat, 2TB RAID 0 attached via USB, very old Caldigit drives from 2006)

I decided not all storage can be Longhorn and attached directly to the PIs in the K3s cluster, so I added a new NFS server with some very old, but gently used HDDs. 

The drive is formatted as EXT4
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

I am removing https://github.com/carlosedp/cluster-monitoring by https://github.com/carlosedp becuase there is very little value here compared to kube-prometheus-stack https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md (which has full multiarch support now).  I am only missing out on the rpi metrics like temperature but I think I can work in the arm-exporter Carlos wrote into my prom stack setup just like I am able to work in speedtest-exporter that comes from https://github.com/MiguelNdeCarvalho/speedtest-exporter by https://github.com/MiguelNdeCarvalho 

To install kube-prometheus-stack all you need to do is:
- helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
- helm repo update
- helm install prometheus-stack prometheus-community/kube-prometheus-stack -n mon --create-namespace --values prom-stack-min.yaml
I did override some chart values to use my NFS storage (even though this goes against Prometheus advice to use local storage, but I don't have much local storage and my setup is small so I'll try it and see how it goes).  I also specified some load balacer IPs as well.  One main this to note in the values are these line:
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    podMonitorSelectorNilUsesHelmValues: false
These essentailly tell the prometheus operator to pick up all serviceMonitors and podMonitors that are deployed in K3s (important to note that serviceMonitor and podMonitor are custom resource definitions in K3s).  If you leave the default values, Prometheus will only pickup prometheus selectors and speedtest-exporter isn't one of them.

To install speedtest exporter, curtesy of https://github.com/k8s-at-home/charts
- helm repo add k8s-at-home https://k8s-at-home.com/charts/ 
- helm repo update
- helm install speedtest-exporter k8s-at-home/speedtest-exporter -f speedtest.yaml -n mon

The speedtest.yaml values file has some small tweaks, like time zone but most importantly enabling the podMonitor so Prometheus can scrape the exporter.