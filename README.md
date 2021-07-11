Why and I doing this?  I work in tech and clustering interests me, also if my entire network is going to depend on a rasperberry pi for DNS, ad blocking, and DHCP, I might as well have some redundacy.
Is this a lot of overhead for redundancy, maybe, but again, it interests me, so why not :-) 
Also we all work from home now, we need the network to stay up and be reliable.

This is not a step by step guide but if you have questions feel free to add an issue and I will try and help.  I do presume you have some knowledge of Kubernetes and helm. 
I had no hands on practice when I started this but YouTube and other sources like (https://github.com/geerlingguy), (https://github.com/justmeandopensource) and (https://github.com/timothystewart6) taught me a lot, so thanks to them.  Check out their YouTube Channels too.

My OS of choice is RaspberryPI OS beta 64bit, seems stable and light weight enough for use.  
I Decided to use K3s (https://k3s.io) for the Kubernetes runtime.  It's lightweight and seems to be stable.  Easy to setup and install.

Hardware is 3 8GB PIs with 64GB EVO PLUS SD cards.  3 POE PI Hats and 1 POE switch.  The nodes are configured as 1 K3s Master and 2 K3s workers nodes (Muppet themed lol).  The master will be a worker node as well.  
TODO: add redundacy to master node.  I plan to do this overtime, perfer 2 more master nodes with embedded database for high availability now that etcd is supported in K3s.

For each PI, I flashed the SD with 2021-05-07-raspios-buster-arm64-lite.zip using the Raspnery PI imager tool, I endabled SSH access by dropping in a ssh file into the SD card.

There are some prerequisits before installing K3s.  
1.  Changed the name of each host to be unique by editing /etc/hostname and reboot
2.  Run sudo apt update
3.  run sudo apt full-upgrade
This updates and uprades packages and other OS bits.

Raspberry OS defaults to using nftables instead of iptables. K3S networking features require iptables and do not work with nftables. Follow the steps below to switch configure the OS to use legacy iptables:
4.  sudo iptables -F
5.  sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
6.  sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
7.  sudo reboot
Standard Raspberry OS installations do not start with cgroups enabled. K3S needs cgroups to start the systemd service. cgroups can be enabled by appending cgroup_memory=1 cgroup_enable=memory to /boot/cmdline.txt.

8. sudo nano /boot/cmdline.txt, append cgroup_memory=1 cgroup_enable=memory to the end of the line like so:
console=serial0,115200 console=tty1 root=PARTUUID=58b06195-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory
9. sudo reboot

I installed master node with curl -sfL https://get.k3s.io | sh -
I installed worker nodes with curl -sfL https://get.k3s.io | K3S_URL=https://<myserver>:6443 K3S_TOKEN=<mynodetoken> sh -
You can checkout the quick start guide on the K3s website.  Be sure to replace the values above with your own.  

I prefer to use MetalLB instead of the stock K3s load balancer (Klipper) becuase MetalLB will assign 1 IP address per service for the entire cluster.  
I also prefer to use nginx ingress instead of Traefik.  Both of these swaps are optional.  It's just preference.  If you swap them, you need to start the K3s without these options:

You need to edit the service file on the master only, 
1. sudo nano /etc/systemd/system/k3s.service

e.g., end of the file snippet
 ExecStart=/usr/local/bin/k3s \
    server \
    --disable traefik \
    --disable servicelb \
Then reload and restart the service
2. sudo systemctl daemon-reload
3. sudo systemctl restart k3s.service

Be sure to configure your client machine for kubectl and helm so you can access your cluster, you will need them.  If you are on mac, you can use homebrew to install both.

Now you are ready to install MetalLB as your bare metal load balancer, you will need a block of IP addresses your current DHCP server will not assign to other devices on your network, once you have it you are ready to install MetalLB.
I guess I wasn't too comfotable with Helm yet so I used the manifests on MetalLB's website, https://metallb.org/installation/ 

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml

After you deploy both manifests, you will need to apply a config map using kubectl apply -f somefile.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.0.0.2-10.0.0.11

This is the simplest way to get metallb up and running with a block of addresses for it to hand out to loadbalancer services.  Now metallb is ready and you can test it by deploying an nginx pod and loadbalancer service.
You can further test by deploying 2 nginx pods and 2 LB services to ensure metallb is giving out the IPs correctly. (notes that you can share IPs for a deployment that has more than 1 service like pihole (DNS, Web, DHCP), so 10 addresses seemed like a good number, but we will see)

At this point, I needed a storage solution, I don't own a NAS, so NFS was out of the question, so I went with Longhorn.  Longhorn is a  block storage system that can replicate data around to all the worker nodes in your K3s cluster.  Pretty simple install using helm.  
There are some prerequisits to using longhorn (I had all of them already excpet for iscsi), all hosts that will keep storage replicas (for me this is all 3) will need have open-iscsi installed, it's easy to install, 

sudo apt install open-iscsi.  

The steps to install longhorn are:
1. kubectl create namespace longhorn-system
2. helm install longhorn longhorn/longhorn --namespace longhorn-system

Longhorn doc says in order to acess the frontend UI, you will need have an ingress controller, I don't think this is true, you can just edit the frontend service or pass in some values for the helm chart and set the service to be a LoadBalancer (preferred way).  However, there is no authentication this way.  Then MetalLB will hand out an IP for it.  Full disclosure I did install ingress-nginx using Arkade (https://github.com/alexellis/arkade ... I could have used helm to install ingress-nginx, but I didn't) and configured it per longhorns doc. 

Now we are ready to deploy for the services: I have chosen to start with 3 services
1. Pihole (https://github.com/MoJo2600/pihole-kubernetes) - I used this helm chart to deploy pihole, very nice and credit to https://github.com/MoJo2600 and all the contributors for all their hard work.  I am using pihole as my DHCP and DNS server.  You can access my pihole values yaml file in this repo
helm install --namespace pihole --values pihole.yaml pihole mojo2600/pihole

2. Unbound, I had some work to do here, no helm chart exists that I could find, no kubernetes manifests either.  Just some 4 year old stuff I saw on github.  So I wrote my own manifests based on the great work that https://github.com/MatthewVance has done to provide unbound on arm (https://github.com/MatthewVance/unbound-docker-rpi).  

You can access my unbound manfest files in this repo.  

Another note on Unbound, you can run it stateless but I wanted more control over the config, this is convoluted when it comes to storage.  If you define a volume mount for the container, the unbound-docker-rpi container will presume you have already copied/crearted all Unbound config files to that volume or the container simply won't start.  

With Longhorn this isn't straight forward.  You basically need to mount the virtual disk (volume) that longhorn creates to a node an copy the files to there, then deploy the K3s manifests.  This issue summarizes the issue, https://github.com/longhorn/longhorn/issues/265
See this comment for steps to do this, it worked great for me, https://github.com/longhorn/longhorn/issues/265#issuecomment-770968271

I am debating whether to keep this volume for Unbound.  I really did it so I can keep root.hints up to date, I don't know if I will keep up with it or not.  Might be too much trouble if I start fiddling with the settings and I break the container, then I would need to mount the image and fix it.  A little clunky and I know this because I have already broken it twice adding some settings.

A. kubectl apply -f unbound-pvc.yaml -n pihole
--this is where you want to mound the volume and copy in the config files)
B. kubectl apply -f unbound.yaml -n pihole
C. kubectl apply -f unbound-svc.yaml -n pihole

Yes I know this can all go into one file and namespaces can be hardcoded, I'm lazy.  
Further, since the Unbound container can move around the cluster, the service defintion has the ClusterIP hard coded, it's an internal IP to the cluster.  Make sure no other service has this IP in your cluster.  I wanted to have a "Statically" configured IP so that pihole could always find the Unbound service

3. Home Assistant (https://k8s-at-home.com), credit to this project and all teh 150+ contributors for creating an easy to use helm charts like HA.  You can see my values file in this repo.
I assume by now you know how to use Helm :-)

Failover testing.  I was able to drain a node (kubectl drain kermit --ignore-daemonsets --delete-local-data) that services were running on and watch them failover to another node, there is a short interuption to my network and internet access but this is much better than scrambling to get a new pi online or killing the pihole alltogether and removing it from my setup, even if temporarily.  Also I run a lot of lights off of Home assistant, will be nice not to set that up again in case of hardware failure (pi board, sd card, etc).  Failovers work, longhorn replication works, it's all working good... until it's not :-)

Some things to remind myself of incase of a client computer crash
helm repo list
NAME         	URL                                          
cockroachdb  	https://charts.cockroachdb.com/              
ingress-nginx	https://kubernetes.github.io/ingress-nginx   
mojo2600     	https://mojo2600.github.io/pihole-kubernetes/
metallb      	https://metallb.github.io/metallb            
longhorn     	https://charts.longhorn.io                   
k8s-at-home  	https://k8s-at-home.com/charts/       

helm list -A
NAME          	NAMESPACE      	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
home-assistant	home-assistant 	2       	2021-07-09 15:13:02.391874 -0400 EDT   	deployed	home-assistant-9.2.0	2021.6.3   
ingress-nginx 	default        	1       	2021-07-04 09:55:55.719444 -0400 EDT   	deployed	ingress-nginx-3.34.0	0.47.0     
longhorn      	longhorn-system	1       	2021-07-04 10:04:10.679605 -0400 EDT   	deployed	longhorn-1.1.1      	v1.1.1     
pihole        	pihole         	7       	2021-07-11 15:25:02.966652 -0400 EDT   	deployed	pihole-2.0.0        	           
traefik-crd   	kube-system    	1       	2021-07-04 13:25:37.999827063 +0000 UTC	deployed	traefik-crd-9.18.2  	           

I have no idea what traefik-crd is.  There is nothing in kube-system namespace with that name so could be a remnit from helm itself.  Also Arkade used helm to install ingress-nginx, so in a way I did use helm :-)
