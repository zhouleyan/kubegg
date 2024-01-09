# 通过 kubeadm 快速搭建

```bash
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"

# v1.28.4
RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo wget https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
sudo chmod +x {kubeadm,kubelet}

RELEASE_VERSION="v0.16.2"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable --now kubelet

# The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

cat <<EOF | sudo tee /var/lib/kubelet/config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.28.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

systemctl restart kubelet

# kubeadm config images list

cat >>/etc/hosts<<EOF

# kubegg hosts BEGIN
172.16.0.101 bpf1
172.16.0.102 bpf2
172.16.0.103 bpf3

172.16.0.120 reg.kubegg.io
EOF

kubeadm init --kubernetes-version=v1.28.4 --image-repository reg.kubegg.io/library --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap -v=9
```

```bash
#!/bin/bash
date
set -e
set -v

# 1.prep noCNI env
cat <<EOF | kind create cluster \
--name=cilium-kubeproxy-replacement-ebpf \
--image=reg.kubegg.io/library/node:v1.27.3 \
--config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "172.16.0.130"
  apiServerPort: 16443
  disableDefaultCNI: true
  kubeProxyMode: "none" # Enable KubeProxy
nodes:
- role: control-plane
- role: worker
- role: worker

containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.configs."reg.kubegg.io".tls]
    insecure_skip_verify = true

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."reg.kubegg.io"]
    endpoint = ["https://reg.kubegg.io"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."172.16.0.120"]
    endpoint = ["https://172.16.0.120"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://docker.nju.edu.cn/", "https://kuamavit.mirror.aliyuncs.com"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
    endpoint = ["https://gcr.nju.edu.cn"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
    endpoint = ["https://gcr.nju.edu.cn/google-containers/"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
    endpoint = ["https://quay.nju.edu.cn"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."ghcr.io"]
    endpoint = ["https://ghcr.nju.edu.cn"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."nvcr.io"]
    endpoint = ["https://ngc.nju.edu.cn"]
EOF

# 2.remove taints
controller_node_ip=`kubectl get node -o wide --no-headers | grep -E "control-plane|bpf1" | awk -F " " '{print $6}'`
controller_node=`kubectl get nodes --no-headers  -o custom-columns=NAME:.metadata.name| grep control-plane`
kubectl taint nodes $controller_node node-role.kubernetes.io/control-plane:NoSchedule-
kubectl get nodes -o wide

# 3.change hosts
for i in $(docker ps -a --format "table {{.Names}}" | grep cilium-kubeproxy)
do
  echo $i
  docker exec -it $i bash -c "echo 172.16.0.120 reg.kubegg.io >> /etc/hosts"
done

# 4.install cilium
# Direct Routing Options(--set kubeproxyReplacement=true --set tunnel=disabled --set autoDirectNodeRoutes=true --set ipv4NativeRoutingCIDR="10.0.0.0/8")
# Host Routing[EBPF](--set bpf.masquerade=true)
helm upgrade cilium --install \
--set k8sServiceHost=$controller_node_ip \
--set k8sServicePort=6443 \
--namespace kube-system \
--set debug.enabled=true \
--set debug.verbose=datapath \
--set monitorAggregation=none \
--set ipam.mode=cluster-pool \
--set cluster.name=cilium-kubeproxy-replacement-ebpf \
--set kubeProxyReplacement=true \
--set tunnel=disabled \
--set autoDirectNodeRoutes=true \
--set ipv4NativeRoutingCIDR="10.0.0.0/8" \
-f ../../cilium-values.yaml \
../../cilium-1.14.5.tgz
```
