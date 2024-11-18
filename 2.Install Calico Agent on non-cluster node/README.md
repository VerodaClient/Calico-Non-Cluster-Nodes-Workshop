
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

#### Expose elasticsearch service

```bash
kubectl apply -f-<<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: tigera-secure-es-http-np
  name: tigera-secure-es-http-np
  namespace: tigera-elasticsearch
spec:
  ports:
  - port: 9200
    name: https
    protocol: TCP
    nodePort: 31920
  selector:
    common.k8s.elastic.co/type: elasticsearch
    elasticsearch.k8s.elastic.co/cluster-name: tigera-secure
  type: NodePort
EOF
```

```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: NonClusterHost
metadata:
  name: tigera-secure
spec:
  endpoint: https://10.0.1.30:31920
EOF
```
```bash
kubectl get secret -n calico-system tigera-noncluster-host -o jsonpath='{.data.token}' | base64 --decode
```
```bash
kubectl config view --flatten --minify
```
```bash
apiVersion: v1
kind: Config
current-context: noncluster-hosts
preferences: {}
clusters:
- cluster:
    certificate-authority-data: <your cluster certificate>
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
 profiles:
 - projectcalico-default-allow
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
sudo curl https://downloads.tigera.io/ee/rpms/v3.20/calico_rhel9.repo -o /etc/yum.repos.d/calico_rhel9.repo
```
```bash
sudo dnf install calico-node calico-fluent-bit -y
```

Copy the kubeconfig into /etc/calico/kubeconfig and change ownership to calico:calico

```bash
sudo chown calico:calico /etc/calico/kubeconfig
```

```bash
sudo systemctl enable --now calico-node.service
sudo systemctl enable --now calico-fluent-bit.service
```

Set the node name to be correct for calico-node.env and the node itself:
```bash
echo "NODE_NAME=noncluster" | sudo tee -a /etc/calico/calico-node/calico-node.env
```
```bash
sudo hostnamectl set-hostname noncluster
```
```bash
echo -e "127.0.0.1   localhost noncluster\n::1         localhost noncluster" | sudo tee /etc/hosts
```
```bash
sudo vi /etc/cloud/cloud.cfg
```
change:
```bash
preserve_hostname: false
```
to
```bash
preserve_hostname: true
```

```bash
sudo reboot now
```