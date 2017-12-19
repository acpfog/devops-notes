## How to make a Kubernetes cluster with Kubespray

### Prepare cluster's nodes hardware

Make proper BIOS settings for a standalone work without a keyboard and a monitor.
The BIOS option "After Power Failure" switch to "Power on" and set current time.

### Install and configure OS on the cluster's nodes

Install Ubuntu in a minimal configuration.
Swap can be turned off or set the option `vm.swappiness = 1` in `/etc/sysctl.conf`.
Setup the time synchronization.
Turn off the firewall and SSH bruteforce prevent tools.
Enable ipv4 forwarding on the nodes and they must have access to the Internet.
Install packages:

```console
$ sudo apt-get update
$ sudo apt-get install docker-engine ceph-common ceph-fs-common libcephfs1 python-cephfs --no-install-recommends
```

Note: One of Kubespray's requirements is Docker v1.13, but somehow it works with Docker 17.03.1 on Ubuntu 16.04.3.

### Prepare for a deploy

Create a python virtualenv and switch into it.
Install using pip the latest ansible tool and netaddr.
Clone Kubespray repository and create your inventory file `inventory/inventory.cfg`.
The inventory file contains a list of nodes where Kubespray will be deployed with different roles.
Setup key-based authentication for the user ubuntu:

```console
$ ssh-keygen
$ ssh-copy-id ubuntu@node-01
...
$ ssh-copy-id ubuntu@node-03
```

Grant root privileges to the user ubuntu with sudo.

### Deploy a cluster

```console
$ ansible-playbook cluster.yml -b -i inventory/inventory.cfg -e "cluster_name=cluster.local" -u ubuntu
```

### Access to the cluster with kubectl

Login into the master node where installed kubectl tool.
By default you can connect to the cluster locally using kubectl without authentication.
For instance you can get the status of the cluster nodes:

```console
$ kubectl get nodes
```

Using kubectl you can create a Kubernetes config file and use it outside of the cluster.

```console
$ kubectl config set-cluster my-cluster --server=https://<master_node_ip>:6443 --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true
$ kubectl config set-credentials cluster-admin --certificate-authority=/etc/kubernetes/ssl/ca.pem --client-key=/etc/kubernetes/ssl/admin-node-01-key.pem  --client-certificate=/etc/kubernetes/ssl/admin-node-01.pem --embed-certs=true
$ kubectl config set-context my-cluster-admin --cluster=my-cluster --user=cluster-admin
```

Copy the ~/.kube/config file to other machine where you have installed kubectl and you will able to operate the cluster outside from that place.
But this method is a dirty hack.
Better to use your personal keys and certificates with roles bound to a user (Role Based Access Control).

Check if you have a configured access to you cluster.

```console
$ kubectl config view
```

### Access to the cluster's dashboard

Create a service account for accessing the dashboard

```console
$ kubectl create serviceaccount admin -n default
```

Get a token

```console
$ kubectl describe secrets admin-token -n default
```

Bind the cluster admin role to the new service account

```console
$ kubectl create clusterrolebinding owner-cluster-admin-binding --clusterrole cluster-admin --serviceaccount default:admin -n default
```

Bind the dashboard service to the node port number 30000

```console
$ kubectl patch services kubernetes-dashboard -p '{"spec": {"ports":[{"nodePort": 30000, "port": 443}],"type": "NodePort"}}' -n kube-system
```

Open the dashboard in a browser by URL https://<master_node_ip>:30000 and enter the token

### Monitoring with Heapster

Clone Heapster repository

```console
$ git clone https://github.com/kubernetes/heapster.git
$ cd heapster
```

Add a line

```text
- --sink=influxdb:http://<influxdb_ip_address>:8086?user=influxdb_user_name&pw=influxdb_user_pass&db=database_name
```

at the end of section command: in the file deploy/kube-config/standalone/heapster-controller.yaml

```console
$ kubectl apply -f deploy/kube-config/rbac/heapster-rbac.yaml
$ kubectl apply -f deploy/kube-config/standalone/heapster-controller.yaml
```

### Connect to a private Docker registry

On each cluster's node make a config file with credentials and restart kubelet process.

```console
$ sudo cat <<'EOF' > /var/lib/kubelet/config.json
{
    "auths": {
        "docker.company.local": {
            "auth": "SGVsbG8sIHdvcmxkIQo="
        }
    }
}
EOF

$ sudo systemctl restart kubelet.service
```

### Configure Helm

```console
$ kubectl create serviceaccount tiller -n kube-system
$ kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole cluster-admin --serviceaccount kube-system:tiller -n kube-system
$ helm init --service-account tiller
$ helm ls
```

### Install Rook

```console
$ helm repo add rook-alpha https://charts.rook.io/alpha
$ helm fetch rook-alpha/rook
$ tar zxvf rook-v0.6.1.tgz
$ kubectl create namespace rook
$ helm install --name rook --namespace rook rook

$ cat <<'EOF' > cluster.yaml
apiVersion: rook.io/v1alpha1
kind: Cluster
metadata:
  name: rook
  namespace: rook
spec:
  versionTag: master
  dataDirHostPath: /var/lib/rook4
  storage:
    useAllNodes: true
    useAllDevices: false
    storeConfig:
      storeType: bluestore
      databaseSizeMB: 1024
      journalSizeMB: 1024
EOF

$ cat <<'EOF' > block.yaml
apiVersion: rook.io/v1alpha1
kind: Pool
metadata:
  name: main-block-pool
  namespace: rook
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-block
   annotations:
     storageclass.kubernetes.io/is-default-class: true
provisioner: rook.io/block
parameters:
  pool: main-block-pool
EOF

$ cat <<'EOF' > filesystem.yaml
apiVersion: rook.io/v1alpha1
kind: Filesystem
metadata:
  name: rook
  namespace: rook
spec:
  metadataPool:
    replicated:
      size: 2
  dataPools:
    - replicated:
        size: 2
  metadataServer:
    activeCount: 2
    activeStandby: false
EOF

$ kubectl create -f cluster.yaml
$ kubectl create -f block.yaml
$ kubectl create -f filesystem.yaml
```
### Upgrade cluster

```console
$ ansible-playbook upgrade-cluster.yml -C -b -i inventory/inventory.cfg -e "kube_version=v1.8.4 drain_timeout=600s" -u ubuntu
```
