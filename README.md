Files to easily install K8s into 3 nodes. Can be modified for different environements and cluster sizes

To install all K8s components on all nodes. 
Anything in <> should be replaced with appropriate value

These steps will create a Kubernetes cluster of 3 nodes named control0 for 
the master node and worker0, worker1 for the 2 worker nodes  


$ nano /etc/hosts    on control0 node

add private IP's and node names 

127.0.0.1 localhost

<control0 private IP> control0
<worker0 private IP> worker0
<worker1 private IP> worker1

$ ping -c 4 worker0
  
$ ping -c 4 worker1

$ ssh -A <public host address of worker0>     and modify /etc/hosts

$ exit

$ ssh -A <public host address of worker1>     and modify /etc/hosts

$ exit

These commands are run the control0 node

$ nano ~/.bashrc    and place this line at the bottom of the file

 export PrivateIP=<private Ip address of control0>
    
$ source ~/.bashrc
  
$ echo $PrivateIP

$ git clone -b k8s-v1.28.1 https://github.com/kmarch66/kube-essentials.git

$ cd ~/kube-essentials

~/kube-essentials$ chmod +x ./host-setup.sh

~/kube-essentials$ sudo ./host-setup.sh

~/kube-essentials$ ssh worker0 \
        "sudo bash" < ~/kube-essentials/host-setup.sh

~/kube-essentials$ ssh worker1 \
        "sudo bash" < ~/kube-essentials/host-setup.sh

If you want kubectl completion
~/kube-essentials$ echo 'source <(kubectl completion bash)' \
        | tee --append $HOME/.bashrc

$ . ~/.bashrc

~/kube-essentials$

    $ sudo kubeadm init \
         --kubernetes-version 1.28.1 \
         --apiserver-advertise-address $PrivateIP \
         --pod-network-cidr 192.168.0.0/16

     [init] Using Kubernetes version: v1.28.1
     [preflight] Running pre-flight checks
     ....

~/kube-essentials$ node_join=$(sudo kubeadm token create --print-join-command)

~/kube-essentials$

$ cd ~

$ node_join=$(sudo kubeadm token create --print-join-command)

$ mkdir -p $HOME/.kube

$ sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config

$ sudo chown <user:user> ~/.kube/config       change this to reflect the user running the commands

$ sudo chmod 600 ~/.kube/config

$ kubectl get nodes

 NAME       STATUS     ROLES           AGE    VERSION
 control0   NotReady   control-plane   2m6s   v1.28.1

$ kubectl apply -f ~/kube-essentials/calico.yaml

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

$ ssh worker0 sudo $node_join

    ....
    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

$ ssh worker1 sudo $node_join

    ....
    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

Wait a couple of minutes for cni to make connection -- then run

$ kubectl get nodes 

    NAME       STATUS     ROLES                  AGE     VERSION
    control0   Ready      control-plane,master   18m     v1.28.1
    worker0    Ready      <none>                 4m27s   v1.28.1
    worker1    Ready      <none>                 16s     v1.28.1

The Kubernetes cluster is ready.



To install Helm on the master.

$ sudo curl -O https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz

$ sudo tar xvf helm-v3.16.2-linux-amd64.tar.gz

$ sudo mv linux-amd64/helm /usr/local/bin

$ rm helm-v3.16.2-linux-amd64.tar.gz

$ rm -rf linux-amd64

$ helm --version

Helm is installed

If the user on the control0 node can ssh to the worker nodes use this command

$ chmod +x ~/kube-essentials/nfs-setup.sh ; \
         ~/kube-essentials/nfs-setup.sh

Then jump over the nfs installation section to the helm nfs-client section  
      
         
To install dynamic provisioned nfs storage class on the nodes if the user can not ssh direct from node to node .

On each cluster node run these to install NFS

$ sudo apt update

$ sudo apt install nfs-kernel-server

$ sudo apt install nfs-common

On the control0 node run these commands

$ sudo mkdir /srv/nfs/kubedata -p

$ sudo chown nobody: /srv/nfs/kubedata/

Edit the /etc/exports file 

$ sudo vi /etc/exports

Add this line to exports file
/srv/nfs/kubedata    *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
Save and exit vi

$ sudo systemctl enable nfs-server

$ sudo systemctl start nfs-server

$ sudo systemctl status nfs-server

Export the directory to the worker nodes

$ sudo exportfs -rav

Test the nfs mounts
$ sudo mount -t nfs <ip address of control0>:/srv/nfs/kubedata /mnt

$ mount |grep kubedata 

output looks something like this
   192.168.1.152:/srv/nfs/kubedata on /mnt type nfs4 (rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,namlen=    255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.152,local_lock=none,addr=192.168.1.152)

$ sudo unmount /mnt

NFS installation is complete

Now use a helm chart to automate the nfs client provisioner for kubernetes

$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

$ helm repo update

$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=ip-<ip address of control0> --set nfs.path=/srv/nfs/kubedata

$ kubectl get storageclass     

NAME                 PROVISIONER                            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   
nfs-client           cluster.local/nfs-client-provisioner   Delete          Immediate              true                   

$ kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

NFS is now set up as the default dynamic storage provisioner for Kubernetes
