---
layout: post
title: Kubeadm + containerd
img: kubeadm.png
tags: [kubernetes, containerd]
---

# __Setup a local kubernetes cluster with Kubeadm__

In this post, we will see how to setup a local kubernetes 1.24 cluster bootstrapped using `Kubeadm`.  
The setup will be the following:
- [Containerd](https://github.com/containerd/containerd) as the Container Runtime Interface (CRI)
- [Calico](https://projectcalico.docs.tigera.io/about/about-calico) as the Container Network Interface (CNI)

### __Prerequisites__

Create 3 Virtual Machines:
- 1 control plane node: Ubuntu 22.04, 2 CPU and 4096 MB RAM
- 2 worker nodes: Ubuntu 22.04, 2 CPU and 2048 MB RAM

## __The Why__

First of all why setup a local cluster with `kubeadm` over other tools like Minikube, Kind, K3S, microk8s, etc... ?  
I have three reasons for wanting do that for my learning purpose:
- Kubeadm provides a full production level architecture
- It lets you choose the container runtime of your choice
- Kubeadm itself is part of kubernetes certification exams (CKA and CKS)

## __Step 1: Prepare all nodes__

__All this step should be executed on all nodes.__

First, we will update and install some packages that we will need later
```sh
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl apt-transport-https libseccomp2
```

`Swap` should be disabled in order for kubelet to work properly.  
Since Kubernetes 1.8, a kubelet flag `fail-on-swap` has been set to `true` by default, that means swap is not supported by default.  
More more information on the Why, there is a [great article](https://leizhilong.github.io/post/why-swap-should-be-disabled-on-kubernetes/)

```sh
# disable swap
sudo sed -i "/ swap / s/^/#/" /etc/fstab
sudo swapoff -a
```

We also need to let IPtables see bridged traffic
```sh
# Configure IPTables to see bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s-cri-containerd.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl without reboot
sudo sysctl --system
```

Now, let's setup `Containerd`

```sh
# Install containerd as a service
curl -LO https://github.com/containerd/containerd/releases/download/v1.6.6/containerd-1.6.6-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.6-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo cp containerd.service /lib/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

# Create containerd configuration
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable Cgroup
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Systemd drop-in for containerd
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/0-containerd.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
EOF

sudo systemctl daemon-reload
sudo systemctl start containerd
```

Finally, let's install `Kubeadm`, `Kubelet` and `Kubectl`
```sh
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.24.2-00 \
                        kubeadm=1.24.2-00 \
                        kubectl=1.24.2-00
sudo apt-mark hold kubelet=1.24.2-00 kubeadm=1.24.2-00 kubectl=1.24.2-00
```

## __Step 2: Setup the control plane node__

We will run the `kubeadm init` command on the control plane. We will use the following CIDR `192.168.0.0/16` for the pod network needed by Calico CNI that will be installed later on.

Note: The IP adress I assigned to my control plane is the following `10.0.0.10`

```sh
#Init Kubeadm
sudo kubeadm init --apiserver-advertise-address="10.0.0.10" --apiserver-cert-extra-sans="10.0.0.10" --pod-network-cidr="192.168.0.0/16" --node-name=$(hostname -s)
```

We need to retrieve the kubeconfig file.  
```sh
cp /etc/kubernetes/admin.conf ~/.kube/config
```
You can also copy the file to your local machine to be able to access the cluster from it.

Now let's install `Calico`
```sh
# Install Calico
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

And also the metrics server
```sh
# Install metrics server
curl -OL https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
sed -i "/--metric-resolution/a\        - --kubelet-insecure-tls" components.yaml
kubectl apply -f components.yaml
rm components.yaml
```

Finally, we need to retrieve the `kubeadm token` to be able to `join` worker nodes to the control plane. Keep the output, we will use it just after.
```sh
kubeadm token create --print-join-command
```

## __Step 3: Setup worker nodes__

__All this step should be executed on all worker nodes.__

First, copy the kubeconfig file from the controle plane node (~/.kube/config) to your local machine and worker nodes (put it also in ~/.kube/config).

We just need to run the `kubeadm join` command we retrieve ealier in order to add the worker nodes to the cluster.

```sh
# The join command is the output of the kubeadm token command previously executed
kubeadm join 10.0.0.10:6443 --token ggzo41.07ycxprqxi04msXY --discovery-token-ca-cert-hash sha256:b2b4fe7994327c32edb226ec843396391fb3674ab8970f558873fc34b2ed669b
```

We will also add a label on worker nodes
```sh
kubectl label node $(hostname -s) node-role.kubernetes.io/worker=worker-node
```

We have now setup a kubernetes 1.23 cluster with kubeadm using `containerd` and `calico`.
```sh
kubectl get nodes

NAME            STATUS   ROLES                  AGE   VERSION
master-node     Ready    control-plane,master   10m   v1.23.5
worker-node01   Ready    worker                  2m   v1.23.5
worker-node02   Ready    worker                  1m   v1.23.5
``` 
On last step we will restart coredns and metrics server.
```sh
kubectl rollout restart deploy/coredns -n kube-system
kubectl rollout restart deploy metrics-server -n kube-system
```

## __Step 4: Test our cluster with Nginx__

If you didn't do it before, copy the kubeconfig file from the controle plane node (~/.kube/config) to your local machine. Or you can execute the command directly from a node.

Run and expose as `NodePort` Nginx to test our cluster.
```sh
kubectl run nginx --image=nginx
pod/nginx created

kubectl expose pod nginx --type=NodePort --port 80
service/nginx exposed

kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          28s

kubectl get svc
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)       AGE
nginx   NodePort   10.97.234.217   <none>        80:31062/TCP  42s
```

We can now access the service using node IP on port 31062. (Make sure your firewall rules are properly set)

```sh
curl http://10.0.0.10:31062

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## __Step 5: Automation__

In order to create/destroy clusters on demand, I did some automation with [Vagrant](https://www.vagrantup.com/downloads) using virtualBox and bash script.  
Everything is on [my github](https://github.com/ludovicalarcon/kubeadm-vagrant-cluster)
```sh
cd kubeadm-vagrant-cluster
# To create a cluster
vagrant up
# To destroy a cluster
vagrant destroy
```