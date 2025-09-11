README.md

Commands to install K8s w/helm, nfs dynamic storage and external access to the components.  Can be modified for different environments and cluster sizes


All commands to be run will be prefaced by a prompt $, do not include this if copy/paste is used  
Anything in <> should be replaced with appropriate value, nano can replaced with your preferred editor.

These steps will create a Kubernetes cluster of 3 nodes 

On the control0 node
$ nano /etc/hosts    

Add private IP's and node names to the /etc/hosts file it should look something like this when finished

127.0.0.1 localhost

<control0 private IP> control0

<node1 private IP> node1

<node2 private IP> node2

<node3 private IP> node3

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


$ ping -c 4 node1
$ ping -c 4 node2
$ ping -c 4 node3
$ ssh -A <private host address of node1>     and modify /etc/hosts
$ exit
$ ssh -A <private host address of node2>     and modify /etc/hosts
$ exit
$ ssh -A <private host address of node3>     and modify /etc/hosts
$ exit

These commands are run the control0 node

$ nano ~/.bashrc    and place this line at the bottom of the file

 export PrivateIP=<private Ip address of control0>
    
$ source ~/.bashrc
  
$ echo $PrivateIP

$ git clone -b k8s-v1.28.1 https://github.com/kmarch66/kube-setup.git

$ cd ~/kube-setup

~/kube-setup$ chmod +x ./host-setup.sh
~/kube-setup$ sudo ./host-setup.sh

~/kube-setup$ ssh node1 \
        "sudo bash" < ~/kube-setup/host-setup.sh

~/kube-setup$ ssh node2 \
        "sudo bash" < ~/kube-setup/host-setup.sh

~/kube-setup$ ssh node3 \
        "sudo bash" < ~/kube-setup/host-setup.sh

If you want kubectl completion
~/kube-setup$ echo 'source <(kubectl completion bash)' \
        | tee --append $HOME/.bashrc

$ . ~/.bashrc

~/kube-setup$

    $ sudo kubeadm init \
         --kubernetes-version 1.28.1 \
         --apiserver-advertise-address $PrivateIP \
         --pod-network-cidr 192.168.0.0/16

     [init] Using Kubernetes version: v1.28.1
     [preflight] Running pre-flight checks
     ....
     kubeadm join 10.0.141.63:6443 --token 7ca5ge.3i1yfz1ouflty1i5 \
	--discovery-token-ca-cert-hash sha256:76d1ee76273444520a2eadea9a6a7cb68482494b7efcb04c2716

At the finish of the kubeadm init command there will be output that looks like the above with a different ip address, token and token cert hash for your cluster.  Copy this, we will need it later.

Change back to the home directory    
$ cd ~
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
$ sudo chown ubuntu:ubuntu ~/.kube/config       change this to reflect the user running the commands
$ sudo chmod 600 ~/.kube/config
$ kubectl get nodes

 NAME       STATUS     ROLES           AGE    VERSION
 control0   NotReady   control-plane   2m6s   v1.28.1

$ kubectl apply -f ~/kube-setup/calico.yaml

    configmap/calico-config created
    ....
    clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
    clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
    clusterrole.rbac.authorization.k8s.io/calico-node created
    clusterrolebinding.rbac.authorization.k8s.io/calico-node created
    daemonset.apps/calico-node created
    serviceaccount/calico-node created
    deployment.apps/calico-kube-controllers created
    serviceaccount/calico-kube-controllers created

$ kubectl get nodes    it may take a minute or 2 for control0 to return Ready status

    NAME       STATUS     ROLES                  AGE     VERSION
    control0   Ready      control-plane,master    2m     v1.28.1

$ ssh node1 \
 sudo kubeadm join 10.0.141.63:6443 --token 7ca5ge.3i1yfz1ouflty1i5 \
	--discovery-token-ca-cert-hash sha256:76d1ee76273444520a2eadea9a6a7cb68482494b7efcb04c2716 

$ ssh node2 \
 sudo kubeadm join 10.0.141.63:6443 --token 7ca5ge.3i1yfz1ouflty1i5 \
	--discovery-token-ca-cert-hash sha256:76d1ee76273444520a2eadea9a6a7cb68482494b7efcb04c2716 

$ ssh node3 \
 sudo kubeadm join 10.0.141.63:6443 --token 7ca5ge.3i1yfz1ouflty1i5 \
	--discovery-token-ca-cert-hash sha256:76d1ee76273444520a2eadea9a6a7cb68482494b7efcb04c2716 
$ exit


Wait a couple of minutes for cni to make connection -- then run

$ kubectl get nodes 

    NAME       STATUS     ROLES                  AGE     VERSION
    control0   Ready      control-plane,master   18m     v1.28.1
    node1      Ready      <none>                 4m27s   v1.28.1
    node2      Ready      <none>                 2m10s   v1.28.1
    node3      Ready      <none>                 16s     v1.28.1
You may see that K8s named your nodes by the AWS IP names instead of your host names

The Kubernetes cluster is ready.

To install Helm on control0.

$ sudo curl -O https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
$ sudo tar xvf helm-v3.16.2-linux-amd64.tar.gz
$ sudo mv linux-amd64/helm /usr/local/bin
$ sudo rm helm-v3.16.2-linux-amd64.tar.gz
$ sudo rm -rf linux-amd64
$ helm version
 
Helm is installed

If the user on the control0 node can ssh to the worker nodes use this command, you will need to alter the nfs-setup.sh if your cluster names and number of nodes differ from the ones used here

$ chmod +x ~/kube-setup/nfs-setup.sh ; \
         ~/kube-setup/nfs-setup.sh

Then jump over the nfs package installation section   
      
         
To install nfs packages on the nodes if the user can not run the above command .

On each cluster node run these commands to install NFS
$ sudo apt update
$ sudo apt install nfs-kernel-server
$ sudo apt install nfs-common

On the control0 node run these commands

$ sudo mkdir /srv/nfs/kubedata -p
$ sudo chown nobody: /srv/nfs/kubedata/

Edit the /etc/exports file 
$ sudo nano /etc/exports

Add this line to exports file
/srv/nfs/kubedata    *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
Save and exit vi

$ sudo systemctl enable nfs-server
$ sudo systemctl start nfs-server
$ sudo systemctl status nfs-server

Export the directory to the worker nodes 
$ sudo exportfs -rav

Test the nfs mounts
$ sudo mount -t nfs <Privateip address of control0>:/srv/nfs/kubedata /mnt
$ mount |grep kubedata 

output looks something like this
   192.168.1.152:/srv/nfs/kubedata on /mnt type nfs4 (rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,namlen=    255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.152,local_lock=none,addr=192.168.1.152)

$ sudo umount /mnt

NFS installation is complete

Now use a helm chart to automate the nfs client provisioner for kubernetes

$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm repo update
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=<ip address of control0> --set nfs.path=/srv/nfs/kubedata
$ kubectl get storageclass     

NAME                 PROVISIONER                            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   
nfs-client           cluster.local/nfs-client-provisioner   Delete          Immediate              true                   

$ kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

NFS is now set up as the default dynamic storage provisioner for Kubernetes

