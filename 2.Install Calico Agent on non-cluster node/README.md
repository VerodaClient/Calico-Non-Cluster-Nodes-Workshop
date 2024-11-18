
# In this lab

This lab provides the instructions to:

* [Overview](xxx)
* [Deploy Calico agent on non-cluster node](xxx)



### Overview

Not all hosts in your environment run pods/workloads. You may have physical machines or legacy applications that you cannot move into a Kubernetes cluster, but still need to securely communicate with pods in your cluster. Calico Enterprise lets you enforce policy on these non-cluster hosts using the same robust Calico Enterprise network policy that you use for pods. This solution can also be used to protect bare metal/physical servers that run Kubernetes clusters instead of VMs.


#### Documentation

- https://docs.tigera.io/calico-enterprise/latest/getting-started/bare-metal/about

____________________________________________________________________________________________________________________________________________________________________________________


### Deploy Calico agent on non-cluster node

#### Non-cluster architecture

A non-cluster host is a computer that is running an application that is not part of a Kubernetes cluster. But you can protect hosts using the same Calico Enterprise network policy that you use for your Kubernetes cluster. In the following diagram, the Kubernetes cluster is running full Calico Enterprise with networking (for pod-to-pod communications) and network policy; the non-cluster host uses Calico Enterprise network policy only for host protection.

<image>

For non-cluster hosts, you can secure host interfaces using host endpoints. Host endpoints can have labels, and work the same as labels on pods/workload endpoints. The advantage is that you can write network policy rules to apply to both workload endpoints and host endpoints using label selectors; where each selector can refer to the either type (or be a mix of the two). For example, you can write a cluster-wide policy for non-cluster hosts that is immediately applied to every host.

#### Set up your Kubernetes cluster to work with a non-cluster host or VM

```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: NonClusterHost
metadata:
  name: tigera-secure
spec:
  endpoint: <endpoint for non-cluster host log forwarder>
EOF
```
```bash
kubectl get secret -n calico-system tigera-noncluster-host -o jsonpath='{.data.token}' | base64 --decode
```

```bash
apiVersion: v1
kind: Config
current-context: noncluster-hosts
preferences: {}
clusters:
- cluster:
    Certificate-authority-data: <your cluster certificate>
    server: <your cluster server>
  name: noncluster-hosts
contexts:
- context:
    cluster: noncluster-hosts
    user: tigera-noncluster-host
  name: noncluster-hosts
users:
- name: tigera-noncluster-host
  user:
    token: <token from previous step>
```
```bash
kubectl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
 name: non-cluster-node
 labels:
   noncluster: "true"
spec:
 interfaceName: eth0
 node: noncluster
 expectedIPs: ["10.0.1.222"]
EOF
```

#### Set up your non-cluster host or VM

ssh to non-cluster node
```bash
ssh noncluster
```

##### Pre-requisited
Install ipset
```bash
sudo yum install ipset -y
```

Install conntrack
```bash
sudo yum install conntrack -y
```

```bash
curl https://downloads.tigera.io/ee/rpms/v3.20/calico_rhel9.repo -o /etc/yum.repos.d/calico_rhel9.repo
```
```bash
dnf install calico-node calico-fluent-bit
```

Copy the kubeconfig into /etc/calico/kubeconfig and change ownership to calico:calico

```bash
chown calico:calico /etc/calico/kubeconfig
```

```bash
systemctl enable --now calico-node.service
systemctl enable --now calico-fluent-bit.service
```

Optionally change some variables in /etc/calico/calico-node/calico-node.env