# Homelab Cluster Setup

## Creating the cluster

On Ubuntu 22.04 machines:

### Disable swap:

`swapoff -a`

### Install containerd from docker:

https://docs.docker.com/engine/install/ubuntu/

## Configure containerd:
copy [config.toml](resources/config.toml) to /etc/containerd/config.toml

### Install kubeadm:

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Configure crictl:

Copy [crictl.yaml](resources/crictl.yaml) to /etc/crictl.yaml

### Create the master

Get the [kubeadm-config.yaml](resources/kubeadm-config.yaml) file

Run `kubeadm init --config kubeadm-config.yaml`

The kubeconfig file is in `/etc/kubernetes/admin.conf`

#### Connect nodes

You can print the command for the nodes again with `kubeadm token create --print-join-command`

### Install Calico CNI plugin

#### On the master:

From https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/install-cni-plugin

```
openssl req -newkey rsa:4096 \
           -keyout cni.key \
           -nodes \
           -out cni.csr \
           -subj "/CN=calico-cni"
```

```
sudo openssl x509 -req -in cni.csr \
                  -CA /etc/kubernetes/pki/ca.crt \
                  -CAkey /etc/kubernetes/pki/ca.key \
                  -CAcreateserial \
                  -out cni.crt \
                  -days 365

sudo chown $(id -u):$(id -g) cni.crt
```

Create a cni.kubeconfig file:

```
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')

kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=$APISERVER \
    --kubeconfig=cni.kubeconfig

kubectl config set-credentials calico-cni \
    --client-certificate=cni.crt \
    --client-key=cni.key \
    --embed-certs=true \
    --kubeconfig=cni.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=calico-cni \
    --kubeconfig=cni.kubeconfig

kubectl config use-context default --kubeconfig=cni.kubeconfig
```
Provision RBAC:

```
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-cni
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
 # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
      - clusterinformations
      - ippools
    verbs:
      - get
      - list
EOF
```

Bind the cluster role to the calico-cni account

`kubectl create clusterrolebinding calico-cni --clusterrole=calico-cni --user=calico-cni`

#### On all nodes

**Run these as root**

Install binaries:
```
curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-amd64

chmod 755 /opt/cni/bin/calico

curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-ipam-amd64

chmod 755 /opt/cni/bin/calico-ipam
```

Create config directory:

`mkdir -p /etc/cni/net.d/`

Copy the kubeconfig from above:
```
cp cni.kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```

Write the CNI configuration:

```
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

### Install Tigera Operator

From https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises

Install the operator:

`kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml`

Download the CRDs:

`curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml -O`

Modify the CRDs to match your ip range

`kubectl create -f custom-resources.yaml`

### Approve CSRs

`k get csr`

`k certificate approve $CSR`

### Install metrics server