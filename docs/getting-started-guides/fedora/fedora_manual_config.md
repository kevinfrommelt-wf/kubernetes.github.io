---
approvers:
- aveshagarwal
- eparis
- thockin
title: Fedora (Single Node)
---

* TOC
{:toc}

## Prerequisites

1. You need 2 or more machines with Fedora installed. These can be either bare metal machines or virtual machines.

## Instructions

This is a getting started guide for Fedora.  It is a manual configuration so you understand all the underlying packages / services / ports, etc...

This guide will only get ONE node (previously minion) working.  Multiple nodes require a functional [networking configuration](/docs/concepts/cluster-administration/networking/) done outside of Kubernetes.  Although the additional Kubernetes configuration requirements should be obvious.

The Kubernetes package provides a few services: kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy.  These services are managed by systemd and the configuration resides in a central location: /etc/kubernetes.  We will break the services up between the hosts.  The first host, fed-master, will be the Kubernetes master.  This host will run the kube-apiserver, kube-controller-manager, and kube-scheduler.  In addition, the master will also run _etcd_ (not needed if _etcd_ runs on a different host but this guide assumes that _etcd_ and Kubernetes master run on the same host).  The remaining host, fed-node will be the node and run kubelet, proxy and docker.

**System Information:**

Hosts:

```conf
fed-master = 192.168.121.9
fed-node = 192.168.121.65
```

**Prepare the hosts:**

* Install Kubernetes on all hosts - fed-{master,node}.  This will also pull in docker. Also install etcd on fed-master.  This guide has been tested with Kubernetes-0.18 and beyond.
* Running on AWS EC2 with RHEL 7.2, you need to enable "extras" repository for yum by editing `/etc/yum.repos.d/redhat-rhui.repo` and changing the `enable=0` to `enable=1` for extras.

```shell
dnf -y install kubernetes
```

* Install etcd

```shell
dnf -y install etcd
```

* Add master and node to /etc/hosts on all machines (not needed if hostnames already in DNS). Make sure that communication works between fed-master and fed-node by using a utility such as ping.

```shell
echo "192.168.121.9    fed-master
192.168.121.65    fed-node" >> /etc/hosts
```

* Edit /etc/kubernetes/config (which should be the same on all hosts) to set
the name of the master server:

```shell
# Comma separated list of nodes in the etcd cluster
KUBE_MASTER="--master=http://fed-master:8080"
```

* Disable the firewall on both the master and node, as docker does not play well with other firewall rule managers.  Please note that iptables-services does not exist on default fedora server install.

```shell
systemctl disable iptables-services firewalld
systemctl stop iptables-services firewalld
```

**Configure the Kubernetes services on the master.**

* Edit /etc/kubernetes/apiserver to appear as such.  The service-cluster-ip-range IP addresses must be an unused block of addresses, not used anywhere else.  They do not need to be routed or assigned to anything.

```shell
# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://127.0.0.1:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# Add your own!
KUBE_API_ARGS=""
```

* Edit /etc/etcd/etcd.conf to let etcd listen on all available IPs instead of 127.0.0.1. If you have not done this, you might see an error such as "connection refused".

```shell
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
```

* Start the appropriate services on master:

```shell
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

* Addition of nodes:

* Create following node.json file on Kubernetes master node:

```json
{
    "apiVersion": "v1",
    "kind": "Node",
    "metadata": {
        "name": "fed-node",
        "labels":{ "name": "fed-node-label"}
    },
    "spec": {
        "externalID": "fed-node"
    }
}
```

Now create a node object internally in your Kubernetes cluster by running:

```shell
$ kubectl create -f ./node.json

$ kubectl get nodes
NAME            STATUS        AGE      VERSION
fed-node        Unknown       4h
```

Please note that in the above, it only creates a representation for the node
_fed-node_ internally. It does not provision the actual _fed-node_. Also, it
is assumed that _fed-node_ (as specified in `name`) can be resolved and is
reachable from Kubernetes master node. This guide will discuss how to provision
a Kubernetes node (fed-node) below.

**Configure the Kubernetes services on the node.**

***We need to configure the kubelet on the node.***

* Edit /etc/kubernetes/kubelet to appear as such:

```shell
###
# Kubernetes kubelet (node) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=fed-node"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://fed-master:8080"

# Add your own!
#KUBELET_ARGS=""
```

* Start the appropriate services on the node (fed-node).

```shell
for SERVICES in kube-proxy kubelet docker; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

* Check to make sure now the cluster can see the fed-node on fed-master, and its status changes to _Ready_.

```shell
kubectl get nodes
NAME            STATUS      AGE      VERSION
fed-node        Ready       4h
```

* Deletion of nodes:

To delete _fed-node_ from your Kubernetes cluster, one should run the following on fed-master (Please do not do it, it is just for information):

```shell
kubectl delete -f ./node.json
```

*You should be finished!*

**The cluster should be running! Launch a test pod.**

## Support Level


IaaS Provider        | Config. Mgmt | OS     | Networking  | Docs                                              | Conforms | Support Level
-------------------- | ------------ | ------ | ----------  | ---------------------------------------------     | ---------| ----------------------------
Bare-metal           | custom       | Fedora | _none_      | [docs](/docs/home/fedora/fedora_manual_config)            |          | Project

For support level information on all solutions, see the [Table of solutions](/docs/home/#table-of-solutions) chart.

