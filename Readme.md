# Kubeedge Lab

## Quickstart

Coming soon !

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
export CONTAINERD_VER=1.3.1
curl -Lo /tmp/containerd.tar.gz "https://storage.googleapis.com/cri-containerd-release/cri-containerd-${CONTAINERD_VER}.linux-amd64.tar.gz"
sudo tar -C / -xzf /tmp/containerd.tar.gz
sudo systemctl start containerd
sudo systemctl enable containerd
rm /tmp/containerd.tar.gz
```

We need to install [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) to bootstrap our Kubernetes cluster:

```bash
sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y kubelet=1.16.3-00 kubeadm=1.16.3-00 kubectl=1.16.3-00
sudo apt-mark hold kubelet kubeadm kubectl
```

Install the Kubernetes cluster:

```bash
sudo kubeadm init --kubernetes-version=1.16.3 --skip-token-print
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
kubectl apply -f manifests/calico.yaml
```

### Install the Cloud Part

Install the CRDs for kubedge (we are still von cloudcore):

```bash
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/crds/devices/devices_v1alpha1_device.yaml
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/crds/devices/devices_v1alpha1_devicemodel.yaml
```

Prepare the certificates for the cloudcore:

```bash
curl -sLO https://raw.githubusercontent.com/kubeedge/kubeedge/v1.1.0/build/tools/certgen.sh
chmod +x certgen.sh
sudo sed -i 's/RANDFILE/#RANDFILE/g' /etc/ssl/openssl.cnf
sudo ./certgen.sh buildSecret | tee ./manifests/cloudcore/06-secret.yaml
```

Now we can install the `kubeedge` `cloudcore` in Kubernetes:

```bash
kubectl apply -f manifests/cloudcore/
```

Validate the installation:

```bash
kubectl -n kubeedge get deploy,svc
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cloudcore   1/1     1            1           36s

NAME                TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)           AGE
service/cloudcore   NodePort   10.100.30.4   <none>        10000:30000/TCP   36s
```

### Prepare files for edgecore

```bash
kubectl apply -f manifests/edgecore/node.yaml
```

Now we need to pull the certificates for the `edgecore`:

```bash
mkdir -p manifests/edgecore/certs
kubectl -n kubeedge get secret cloudcore -o jsonpath={.data."edge\.crt"} | base64 -d > manifests/edgecore/certs/edge.crt
kubectl -n kubeedge get secret cloudcore -o jsonpath={.data."edge\.key"} | base64 -d > manifests/edgecore/certs/edge.key
kubectl -n kubeedge get secret cloudcore -o jsonpath={.data."rootCA\.crt"} | base64 -d > manifests/edgecore/certs/rootCA.crt
```

## Setting up the edgecore

Jump into the `edgenode` machine: `vagrant ssh edgenode`.
In the first place we need to download the required binaries:

Install the prerequisite:

We also need to install `conntrack` adn [docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/):

```bash
sudo apt update
sudo apt install -y conntrack docker.io
```

### EdgeCore

```bash
curl -Lo /tmp/kubeedge.tar.gz https://github.com/kubeedge/kubeedge/releases/download/v1.1.0/kubeedge-v1.1.0-linux-amd64.tar.gz
tar xvfz /tmp/kubeedge.tar.gz -C /tmp
```

```bash
sudo mkdir -p /etc/kubeedge/conf
# Move all files into place
sudo mv /tmp/kubeedge-v1.1.0-linux-amd64/edge/edgecore /etc/kubeedge
sudo mv /tmp/kubeedge-v1.1.0-linux-amd64/edge/conf/* /etc/kubeedge/conf
```

Update the edge configuration:

```bash
sudo sed -i 's/fb4ebb70-2783-42b8-b3ef-63e2fd6d242e/edgenode/g' /etc/kubeedge/conf/edge.yaml
sudo sed -i 's/interface-name:.*/interface-name: enp0s8/g' /etc/kubeedge/conf/edge.yaml
sudo sed -i 's#wss://0.0.0.0:10000#wss://172.2.0.2:30000#g' /etc/kubeedge/conf/edge.yaml
```

Copy the certificates from the `cloudcore` onto the `edgecore`:

```bash
sudo mkdir -p /etc/kubeedge/certs/
sudo cp manifests/edgecore/certs/* /etc/kubeedge/certs/
```

Create a systemd file for the `edgecore`:

```bash
cat << 'EOF' | sudo tee /etc/systemd/system/edgecore.service
[Unit]
Description=edgecore.service

[Service]
Type=simple
ExecStart=/etc/kubeedge/edgecore

[Install]
WantedBy=multi-user.target
EOF
```

Reload the daemon and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart edgecore
sudo systemctl enable edgecore
# Check the status of the edgecore
sudo systemctl status edgecore
```

## Test setup

On the `cloudcore` machine validate that the `edgenode` came up:

```bash
kubectl get nodes
```

Let's create a simple nginx deployment:

```bash
kubectl apply -f manifests/demo/nginx.yaml
```

Check if the Pod is successfully scheduled and started:

```bash
$ kubectl get po -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-5944d75f-sctkh   1/1     Running   0          34s   172.17.0.2   edgenode   <none>           <none>
```

Verify that we can access the nginx pod:

```bash
curl -I 172.2.0.3
HTTP/1.1 200 OK
Server: nginx/1.15.12
Date: Tue, 19 Nov 2019 18:23:21 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
Connection: keep-alive
ETag: "5cb5d3c3-264"
Accept-Ranges: bytes
```
