I would be covering how to go about upgrading a cluster for both traditional Kubernetes cluster and Amazon EKS cluster, I would also be detailing considerations to have in place before doing either of the upgrades
Traditional Kubernetes cluster upgrade
An important consideration to make when upgrading your cluster is API depreciations(this has to do with old API versions which are no longer in use), if you have depreciated API versions, you will have to change your YAML manifest files and clients to reference the new APIs(recommended API version is v1), check Deprecated API Migration Guide | Kubernetes

The upgrade would be for both Master and worker nodes starting with the Master node; upgrading kubeadm, kubelet and kubectl.

Check cluster version

After confirming API versions matches recommended for cluster upgrade to be done, check your Kubernetes version:

kubectl get nodes
This would highlight the master and worker nodes and their versions

Kubeadm version

To go ahead with upgrade we would need to know the the version of kubeadm running on the cluster, we get that by running this command on the Master node:

kubeadm version -o json
Master Node

Next, we get the kubeadm upgrade plan:

sudo kubeadm upgrade plan
Then get the list of available Kubeadm versions with:

sudo apt update
sudo apt-cache madison kubeadm | tac
Unhold kubeadm and install required version

The reason for holding kubeadm is to prevent upgrades, so we run the following(we used version 1.24.6 for the command below, you can change to that you planned upgrading to):

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.24.6-00 && \
sudo apt-mark hold kubeadm
Apply kubeadm upgrade

After holding kubeadm, we apply the command below to upgrade cluster:

sudo kubeadm upgrade node
Your kubeadm is now upgraded in your cluster, you can check the version again:

kubeadm version -o json
Drain node to evict workloads

To have a smooth upgrade we drain nodes to evict all workloads especially evicted pods occupying space in the cluster and just leave daemonsets:

kubectl drain master-node --ignore-daemonsets
If you have your pods running local, the above command won’t apply for it, you will run this instead:

sudo kubectl drain master-node --ignore-daemonsets --delete-local-data
Upgrade kubelet and kubectl

We would equally unhold kubelet and kubectl, then apply upgrade with the following(we used version 1.24.6 for the command below, you can change to that you planned upgrading to).

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=1.24.6-00 kubectl=1.24.6-00 && \
sudo apt-mark hold kubelet kubectl
The Master Node is fully upgraded, we would do the following next:

Restart services:

sudo systemctl daemon-reload
sudo systemctl restart kubelet
Uncondon the Master node (doing this shows the node is ready to receive workload):

kubectl uncordon master-node
Verify node status:

kubectl get nodes
Since the only changes we have made is on our Master, we would only see an upgrade for it while that of the worker nodes remains same way(which we would upgrade next).

Upgrading worker nodes

It is best practice and advisable to upgrade worker nodes one after the other and not the same time as this helps for load distribution and avoid a total downtime of pods or activity in the cluster.

Unhold kubeadm and install required version

Unhold and upgrade would be done with the following:

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.24.6-00 && \
sudo apt-mark hold kubeadm
Upgrade kubeadm

Upgrade will be done with the following:

sudo kubeadm upgrade node
Thereafter, we drain node(do note that the drain command is only ran from the master and you indicate the worker node to be drained in your command)

kubectl drain worker-node01 --ignore-daemonsets
Upgrade kubelet and kubectl

We upgrade kubelet and kubectl next by unholding them first:

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=1.24.6-00 kubectl=1.24.6-00 && \
sudo apt-mark hold kubelet kubectl
Next, we uncondon worker node(to signal it’s ready to receive load, this command is also ran on the master node and not the worker node)

kubectl uncordon worker-node01
Finally, we confirm our upgrade by checking our nodes on the master

kubectl get nodes
We would note the changes as our master and recently upgraded node has same version, you can apply same process for your other nodes.

You can check if all the kube-system control node pods are running

kubectl get po -n kube-system
Upgrading your Amazon EKS cluster
These is a different upgrade process from the traditional kubernetes upgrade, but endeavor to check for API depreciations with that which match the cluster you are upgrading to(you can check the second paragraph on this article for more details on that)

For this upgrade, we would be illustrating with upgrading from version 1.21 to 1.22

Upgrade Eks control plane

The first step it to upgrade the eks control plane

eksctl upgrade cluster --name=eksworkshop-eksctl
And it outputs

[ℹ]  eksctl version 0.66.0
[ℹ]  using region us-west-2
[ℹ]  (plan) would upgrade cluster "eksworkshop-eksctl" control plane from current version "1.21" to "1.22"
[ℹ]  re-building cluster stack "eksctl-eksworkshop-eksctl-cluster"
[✔]  all resources in cluster stack "eksctl-eksworkshop-eksctl-cluster" are up-to-date
[ℹ]  checking security group configuration for all nodegroups
[ℹ]  all nodegroups have up-to-date configuration
[!]  no changes were applied, run again with '--approve' to apply the changes
Running that command outputs your current cluster and the next cluster it should upgrade to(you can only upgrade sequentially)

You run the command again with (to give a go ahead for the upgrade )— approve

eksctl upgrade cluster --name=eksworkshop-eksctl --approve
The process takes about 25 minutes and you can still use the cluster but would experience some slight service interruptions.

Upgrade EKS core Add ons

When you provision an EKS cluster you get three add-ons that run on top of the cluster and that are required for it to function properly:

kubeproxy
CoreDNS
aws-node (AWS CNI or Network Plugin)
In Updating an Amazon EKS cluster Kubernetes version — Amazon EKS we see that we’ll need to upgrade the kubeproxy and CoreDNS. In addition to performing these steps manually with kubectl as documented there you’ll find that eksctl can do it for you as well.

Since we are using eksctl in the workshop we’ll run the two necessary commands for it to do these updates for us:

eksctl utils update-kube-proxy --cluster=eksworkshop-eksctl --approve

eksctl utils update-coredns --cluster=eksworkshop-eksctl --approve
We can confirm we succeeded with retrieving the versions by running:

kubectl get daemonset kube-proxy --namespace kube-system -

kubectl describe deployment coredns --namespace kube-system | grep Image | cut -d "/" -f 3
Upgrading managed node groups

Before going forward with this upgrade, if you are using a cluster autoscaler on your cluster you would need to kill the process with the command below so it doesn’t provision any service during this upgrade:

kubectl scale deployments/cluster-autoscaler --replicas=0 -n kube-system
Then go ahead with the upgrade by running the following:

eksctl upgrade nodegroup --name=nodegroup --cluster=eksworkshop-eksctl --kubernetes-version=1.22
To monitor the process, you can run this command on another terminal:

kubectl get nodes --watch
You’ll notice the new nodes come up (three one in each AZ), one of the older nodes status would be SchedulingDisabled, then eventually that node go away and another new node come up to replace it till old Nodes have gone away. Then it’ll scale back down from 6 Nodes to the original 3.
