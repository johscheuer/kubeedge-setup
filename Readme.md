# Kubeedge Lab

## Quickstart






## Setting up the cloudcore

### Runtime installation

Jump into the `cloucore` vm which will host Kubernetes and the `kubeedge` `cloudcore`: `vagrant ssh cloudcore`

Install the prerequisite:

```bash
sudo tee -a /etc/modules-load.d/containerd.conf <<'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
sudo tee /etc/sysctl.d/99-kubernetes-cri.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

Now we can install [containerd](https://containerd.io):

```bash
export CONTAINERD_VER=1.3.0
curl -sLo /tmp/containerd.tar.gz "https://storage.googleapis.com/cri-containerd-release/cri-containerd-${CONTAINERD_VER}.linux-amd64.tar.gz"
sudo tar -C / -xzf /tmp/containerd.tar.gz
sudo systemctl start containerd
sudo systemctl enable containerd
rm /tmp/containerd.tar.gz
```

On the cloud install [kubeadm]:

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet=1.16.3-00 kubeadm=1.16.3-00 kubectl=1.16.3-00
sudo apt-mark hold kubelet kubeadm kubectl
```

Install the Kubernetes cluster:

```bash
sudo kubeadm init --kubernetes-version=1.16.3 --skip-token-print --upload-certs
```

After the installation run the following steps to be able tu use `kubectl` as normal user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Untaint the master in order to be able to run the `cloudcore`:

```bash
kubectl taint nodes node-role.kubernetes.io/master- --all
```

And finally install a network plugin:

```bash
kubectl apply -f calico.yaml
```

## Install the Cloud Part

Install the CRDs for kubedge:

```bash
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/crds/devices/devices_v1alpha1_device.yaml
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/crds/devices/devices_v1alpha1_devicemodel.yaml
```

Prepare the certificates for the cloudcore:

```bash
curl -sLO https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/tools/certgen.sh
chmod +x certgen.sh
sudo ./certgen.sh buildSecret | tee ./manifests/cloudcore/06-secret.yaml
#readonly caPath=${CA_PATH:-/etc/kubeedge/ca}
#readonly caSubject=${CA_SUBJECT:-/C=CN/ST=Zhejiang/L=Hangzhou/O=KubeEdge/CN=kubeedge.io}
#readonly certPath=${CERT_PATH:-/etc/kubeedge/certs}
#readonly subject=${SUBJECT:-/C=CN/ST=Zhejiang/L=Hangzhou/O=KubeEdge/CN=kubeedge.io}
```

Now we can install the `kubeedge` `cloudcore`:

```bash
pushd /tmp
curl -sLO https://github.com/kubeedge/kubeedge/releases/download/v1.1.0/kubeedge-v1.1.0-linux-amd64.tar.gz
#curl -sLO https://github.com/kubeedge/kubeedge/releases/download/v1.1.0/checksum_kubeedge-v1.1.0-linux-amd64.tar.gz.txt
tar xvfz kubeedge-v1.1.0-linux-amd64.tar.gz
# Somehow kubeedge require this strange setup
# TODO try to move this to another folder
mkdir -p /etc/kubeedge/conf
sudo mv kubeedge-v1.1.0-linux-amd64/cloud/cloudcore/cloudcore /etc/kubeedge
sudo mv kubeedge-v1.1.0-linux-amd64/cloud/cloudcore/conf/* /etc/kubeedge/conf/
popd
```

```bash
sudo mkdir -p /var/lib/kubeedge
# Changed the kubeconfig path
# todo add extra cloudcore user !
sudo ~/cmd/cloudcore
# Would be nice if we would run this inside of kubernetes
# also we don't want to run this as cluster admin!
```

```bash
cat <<EOF | sudo tee /etc/systemd/system/cloudcore.service
[Unit]
Description=cloudcore.service

[Service]
Type=simple
ExecStart=/etc/kubeedge/cloudcore

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start cloudcore
sudo systemctl enable cloudcore
```



## Setting up the edgecore


```bash
pushd /tmp
curl -sLO https://github.com/kubeedge/kubeedge/releases/download/v1.1.0/edgesite-v1.1.0-linux-amd64.tar.gz
tar xvfz edgesite-v1.1.0-linux-amd64.tar.gz
popd
```

```bash
sudo mkdir -p /etc/kubeedge
todo
https://docs.kubeedge.io/en/latest/setup/setup.html#id2
# https://github.com/kubeedge/kubeedge/tree/master/docs
# https://github.com/kubeedge/kubeedge/tree/master/build/edge
# https://docs.kubeedge.io/en/latest/modules/edge/edged.html
```
