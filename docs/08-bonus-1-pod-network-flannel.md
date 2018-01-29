# Using Flannel for kubernetes networking

In a previous section we configure static node Pod CIDR subnet assignation and routing for each worker node. This is far from perfect, as for each new worker node we add, we had to choose a new Pod CIDR subnet and to staticly route it on each workers node.

In this section we will use the ability of kube-controller-manager to auto-assign Node Pod CIDR for each new worker node registering on kube-apiserver.
We will use [Flannel](https://github.com/coreos/flannel) for reading these assignations and to auto handling the route.
After that adding a new worker node will be automatic.

> There are again [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model. Look at Calico or Kube-router.

> We will use Flannel in host gateway mode. This is the most simple and performant mode as Flannel will use no overlay and only add static routes on top of the current network. 

## Cleaning up

On each worker node clean the static route definition you add earlier.

```
ip route del 10.200.10.0/24 via 10.0.0.11
ip route del 10.200.20.0/24 via 10.0.0.12
ip route del 10.200.30.0/24 via 10.0.0.13
```

Remove the CNI bridge configuration (as we will use a new one with flannel).

```
rm /etc/cni/net.d/10-bridge.conf
```

It is better to stop kubelet on each worker node and to delete the node from kubernetes:

On each worker node:

```
systemctl stop kubelet
```

On one controller node:
```
kubectl delete node wrk1
kubectl delete node wrk2
kubectl delete node wrk3
```


## Configure control plane for auto node CIDR assignation

On each controller node replace the unit file of kube-controller-manager with:


```
cat > /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

And restart it.

```
systemctl daemon-reload
systemctl restart kube-controller-manager
```

## Configure Flannel on each worker node

In this section we will configure flannel on worker node. Flannel talk with the api-server to find the 
current subnet for the host it was running on, and it will also add route to each other worker to form
a "full mesh" pod network.

### Download and install Flannel binary

Download the lastest flannel binary:

```
wget https://github.com/coreos/flannel/releases/download/v0.8.0/flanneld-amd64
```

Install the flannel binary:

```
chmod +x flanneld-amd64 && mv flanneld-amd64 /usr/local/bin/
```

Create the flannel.service systemd unit file:

```
cat > flannel.service <<EOF
[Unit]
Description=Flannel
After=nginx.service
Requires=kubelet.service

[Service]
ExecStartPre=/bin/bash -c "systemctl set-environment NODE_NAME=$(hostname)"
ExecStart=/usr/local/bin/flannel \\
  -kube-api-url http://localhost:8080 \\
  -kube-subnet-mgr
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure Flannel

Flannel will need a little configuration file to work properly, and we also need to add a correct CNI definition for it.

On each worker node:

```
mkdir /etc/kube-flannel/
cat > /etc/kube-flannel/net-conf.json <<EOF
{
 "Network": "10.200.0.0/16",
 "Backend": {
  "Type": "host-gw"
 }
}
EOF
```

Configure CNI for Flannel:



