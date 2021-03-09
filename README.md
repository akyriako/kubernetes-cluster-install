# How to install a Kubernetes Cluster on CentOS 8

- Goal

    After completing all the steps you will have a 3-node Kubernetes Cluster running on CentOS 8 

- What do we need
  
  1. CentOS 8 images. You can find free images here https://www.linuxvmimages.com/images/centos-8/#centos-822004. Be sure to download the minimal version as desktop GUI will not be needed.
  2. 3 Virtual Machines on the virtualization software of your preference. Setup 3 VMs one for master node (4GB, 2 CPUs), and two for workers (2GB, 1 CPU).
  3. You are going to needroot priviledges (obviously), but the aformentioned images are already taking care of that, so read their release notes (You can `just sudo su -` and off you go).

***Master node***

1. Configure hosts and hostnames

Configure the hostname of every box with the following command:

`hostnamectl set-hostname master-01.k8s.rhynosaur.home`

Insert the hostname of each node in /etc/hosts (run as well this command on every server):

`cat <<EOF>> /etc/hosts
192.168.1.133 master-01.k8s.rhynosaur.home
192.168.1.134 worker-01.k8s.rhynosaur.home
192.168.1.135 worker-02.k8s.rhynosaur.home
EOF`

2. Disable SELinux

`setenforce 0`

`sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux`

3. Configure Firewall

4. Enable br_netfilter kernel module
   
*br_netfilter* kernel module is needed from Kubernetes in order the pods across the cluster to be able to communicate with each other.

`modprobe br_netfilter`

`echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables`

5. Disable SWAP

`swapoff -a`

Open the file /etc/fstab and comment out the line that is indicating *swap*. 

> [!TIP]
> The OS images come only with *vi* preinstalled. If you have an alternative editor that you would like to work with, like vim or nano, you can install it manually with yum.

`vi /etc/fstab`

> [!TIP]
> **Don't freak out**:
Press *i* in order to start editing the text. 
When you are done press ESC to exit editing mode, and in order to save & quit, type *:wq* (if you want to discard the changes just type *:q*)

6. Install Docker

First install a bunch of prerequisites:

`yum install -y yum-utils device-mapper-persistent-data lvm2
`

and then add the *docker-ce* repository to the system:

`yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
`

Now you can install *docker-ce* with the following command via yum and wait for it to finish:

`yum install -y docker-ce
`

7. Install Kubernetes

Add the Kubernetes repository to the system:

`cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF`

Install the prerequites packages like kubelet, kubedm and kubectl via yum:

`yum install -y kubelet kubeadm kubectl
`

Wait the installation to finish and then reboot the machine:

`shutdown -r now`

> [!IMPORTANT]
> There is something going on with virtualized CentOS 8 instances, and after rebooting enp0s3 is not coming up right away. Don't take any actions, wait for it and it will come up eventually after some minutes. Surely not optimal especially for production workloads, but unfortunately I haven't figured out a solution yet, so stick with waiting.

Start the necessary services, docker and kubelet:

`systemctl start docker && systemctl enable docker
`
`systemctl start kubelet && systemctl enable kubelet
`

Kubernetes and Docker need to use the same cgroup. Make sure first that Docker is using *cgroupfs* as cgroup-driver:

`docker info | grep -i cgroup
`

and then reconfigure Kubernetes cgroup-driver to *cgroupfs*:

`sed -i 's/cgroup-driver=systemd/cgroup-driver=cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf`

Reload due to configuration changes and restart the kubectl service:

`systemctl daemon-reload
`

`systemctl restart kubelet
`

8. Initialize Kubernetes Cluster

`kubeadm init --apiserver-advertise-address=192.168.1.133 --pod-network-cidr=10.244.0.0/16`

> [!IMPORTANT]
> Use your own IP addresses and CIDRs. The API advertise IPv4 address should point to the IP of your master node. The POD network CIDR is the default one that is used for a flannel network, that we are going to install later on as our network component. That is the default value, if you wish something else don't forget to amend as well the configuration file of flannel with the new CIDR value of your choice.

> [!NOTE]
> There will be additional installation instruction that will demonstrate Kubernetes initialization with other virtual networking solutions: Calico and Weaver. 

Wait till kubeadm-init is over, in if this was successful scroll down to the output of the process and find a line that would look like this:

`kubeadm join 192.168.1.133:6443 --token v8b3p1.xdwz9zafwkf4oc1w --discovery-token-ca-cert-hash sha256:aa557ed289f0db77dc2e80b764e23a37f0a02a06790873b5e42a823188876eb4 
`

> [!IMPORTANT]
> Copy this line and save it somewhere as we are going to need it later while configuring our workers.

9. Get Kubernetes Configuration

Let't tidy up and bring the admin.config in a more cozy place

`mkdir -p $HOME/.kube
`

`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
`

`sudo chown $(id -u):$(id -g) $HOME/.kube/config
`

10. Install Network

For this example we are going to use flannel virtual networking. It is the simplest and Calico with Weaver instructions will follow in separate tutorials

In order to deploy a flannel network to Kubernetes run the following command:

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

Wait for it to finish, it will take some time so either be patient or take a small break. You can periodically check the status of the cluster and the pods with the following commands:

`kubectl get nodes
`
`kubectl get pods --all-namespaces
`

When your master node eventually shows up with as Ready, check the pods' status to make sure all pods instances are Running and that all the necessary pod instances are spawned, including one that contains the text *kube-flannel-ds*, and would be an indication that our flannel virtual network is successufully deployed and running.

10. Add Worker Nodes in the Cluster

Take the command we copied from step 8:

`kubeadm join 192.168.1.133:6443 --token v8b3p1.xdwz9zafwkf4oc1w --discovery-token-ca-cert-hash sha256:aa557ed289f0db77dc2e80b764e23a37f0a02a06790873b5e42a823188876eb4 
`

and run it in every box that will host a worker node. If you are interested having more information for the process or you wish to debug in case something went wrong add the following flag to the statement above and run it again:

`--v=5`

> [!TIP]
> You can run it as many times as you want in case of failure without needing to clean up something before you execute it again.

Wait for it to finish, it will take some time so either be patient or take a small break. You can periodically check the status of the cluster and the pods, from your master, with the following commands:

`kubectl get nodes
`
`kubectl get pods --all-namespaces
`

When all nodes appear in the list and their status turns Ready, you are good to go. You have a Kubernetes cluster.
