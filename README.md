# Phase1: Build Driver (don't do that if NVIDIA publish the offical driver
```
zypper in docker
systemctl enable docker --now
git clone https://gitlab.com/fcrozat/nvidia-driver
cd nvidia-driver/sle15
export DRIVER_VERSION=470.82.01
docker build -t driver:${DRIVER_VERSION}-sles15.3 --build-args DRIVER_VERSION=${DRIVER_VERSION} .

# Export to registry or load on local RKE2 nodes
docker push driver:${DRIVER_VERSION}-sles15.3
# OR
docker save driver:${DRIVER_VERSION}-sles15.3 > driver-${DRIVER_VERSION}-sles15.3.tar
scp xxx.tar rke-node:.
export PATH=$PATH:/var/lib/rancher/rke2/bin
ctr -a /run/k3s/containerd/containerd.sock -n k8s.io images import xxx.tar
ctr -a /run/k3s/containerd/containerd.sock -n k8s.io images tag "import-%{yyyy-MM-dd}" nvcr.io/nvidia/driver:470.82.01-sles15.3
```


# Phase 2: Deploy gpu-operator
```
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed
```

Patch 
```
# Issue: nvidia driver cant compile
# SLE BCI need access to SLE repo for kernel-default and kernel-default-devel.
# In RKE2 (contaienrd) pod is not start as root and can't access /etc/SUSEConnect and /etc/zypp/credentials.d/SCCCredentials
# https://documentation.suse.com/sles/15-SP3/single-html/SLES-container/#sec-customize-prebuild-images
# Workaround : add bind-mount in the nvidia driver daemonset
# Issue: nvidia toolkit not ready
# RKE2 stores containerd socket and config.toml in different directory.
# RKE2 also doesn't allow direct config.toml change. we have to use a Go Tmpl
# https://docs.rke2.io/advanced/#configuring-containerd
# https://thenewstack.io/install-a-nvidia-gpu-operator-on-rke2-kubernetes-cluster/
# Workaround : change ENV to good path and to config.toml.tmpl
# toolkit
# - /run/k3s/containerd/
# - /var/lib/rancher/rke2/agent/etc/containerd/
```

# Phase 3: Configure geometry
```
# Choose your geometry
k describe configmaps -n my-gpu-operator default-mig-parted-config

# Apply your geometry
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  

```

# Phase 4 : Profit

```
k apply -f dcgmproftester-mixed.yaml

k apply -f tf-benchmarks-mixed-3g.yaml

```
