### Check regularly this  Gitlab merge request for SLES15-SP3
- [gitlab.com/nvidia/container-images/driver](https://gitlab.com/nvidia/container-images/driver/-/merge_requests/163)

### Initial Github.com issue is here
- [NVIDIA/gpu-operator/issues/308](https://github.com/NVIDIA/gpu-operator/issues/308)

### Source
- [nvidia operator on Gitlab](https://gitlab.com/nvidia/kubernetes/gpu-operator/)
- [nvidia operator on Github](https://github.com/NVIDIA/gpu-operator/)

# Phase1: Build Driver (don't do that when NVIDIA publish the offical driver)
```
zypper in docker
systemctl enable docker --now
git clone https://gitlab.com/fcrozat/nvidia-driver
cd nvidia-driver/sle15
export DRIVER_VERSION=470.82.01
docker build -t driver:${DRIVER_VERSION}-sles15.3 --build-arg DRIVER_VERSION=${DRIVER_VERSION} .


# Export to registry or load on local RKE2 nodes
docker tag driver:${DRIVER_VERSION}-sles15.3 jear/driver:${DRIVER_VERSION}-sles15.3
docker push jear/driver:${DRIVER_VERSION}-sles15.3
# OR
docker save driver:${DRIVER_VERSION}-sles15.3 > driver-${DRIVER_VERSION}-sles15.3.tar
scp xxx.tar rke-node:.
export PATH=$PATH:/var/lib/rancher/rke2/bin
ctr -a /run/k3s/containerd/containerd.sock -n k8s.io images import xxx.tar
ctr -a /run/k3s/containerd/containerd.sock -n k8s.io images tag "import-%{yyyy-MM-dd}" nvcr.io/nvidia/driver:470.82.01-sles15.3
```


# Phase 2: Deploy gpu-operator
```
# Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed

# Single MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=single
```

Patch 
```
# Issue: nvidia driver can't compile
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

# nvidia-container-toolkit-daemonset
spec:
  template:
    spec:
      - args:
        - /usr/local/nvidia
        env:
        - name: RUNTIME_ARGS
          value: --socket /runtime/sock-dir/containerd.sock --config /runtime/config-dir/config.toml.tmpl
...
      - hostPath:
          path: /var/lib/rancher/rke2/agent/etc/containerd/
          type: ""
        name: containerd-config
      - hostPath:
          path: /run/k3s/containerd/
          type: ""
        name: containerd-socket

# nvidia-driver-daemonset
spec:
  template:
    spec:
      containers:
      - args:
        - init
        command:
        - nvidia-driver
        image: jear/driver:470.82.01-sles15.3
        volumeMounts:
        - mountPath: /etc/SUSEConnect
          name: etc-suseconnect
        - mountPath: /etc/zypp/credentials.d/SCCcredentials
          name: vol9
          readOnly: true
      volumes:
        name: crio-hooks
      - hostPath:
          path: /etc/SUSEConnect
          type: ""
        name: etc-suseconnect
      - hostPath:
          path: /etc/zypp/credentials.d/SCCcredentials
          type: ""
        name: vol9
      - hostPath:
          path: /etc/SUSEConnect
          type: ""
        name: etc-suseconnect
      - hostPath:
          path: /etc/zypp/credentials.d/SCCcredentials
          type: ""
        name: vol9

```

# Phase 3: Configure geometry

Examples here : https://github.com/NVIDIA/mig-parted/blob/master/examples/config.yaml

```
# Choose your geometry
k describe configmaps -n my-gpu-operator default-mig-parted-config

# Apply a geometry
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-1g.5gb --overwrite  

# if you are in single mode
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "7",

# if you are in mixed mode
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "1",
  "nvidia.com/mig-1g.5gb.count": "7",


# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  

kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "1",

```

# Phase 4 : Test all-balanced 1g, 2g, 3g

```
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

k apply -f vector-add/jear-vector-add1-1g.yaml && \
k apply -f vector-add/jear-vector-add2-1g.yaml && \
k apply -f dcgmproftester/dcgmproftester-mixed-2g.yaml && \
k apply -f tf-benchmarks/tf-benchmarks-mixed-3g.yaml

```

# Phase 5 : Monitor
```
https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/dcgm-exporter.html#integrating-gpu-telemetry-into-kubernetes

helm repo add prometheus-community \
   https://prometheus-community.github.io/helm-charts
helm repo update 
helm search repo kube-prometheus
helm inspect values prometheus-community/kube-prometheus-stack > kube-prometheus-stack.values.default


sdiff -s kube-prometheus-stack.values.default kube-prometheus-stack.values.jerome 
    type: ClusterIP					      |	    type: LoadBalancer
    type: ClusterIP					      |	    type: LoadBalancer
    serviceMonitorSelectorNilUsesHelmValues: true	      |	    serviceMonitorSelectorNilUsesHelmValues: false
    additionalScrapeConfigs: []				      |	    additionalScrapeConfigs:
							      >	    - job_name: gpu-metrics
							      >	      scrape_interval: 1s
							      >	      metrics_path: /metrics
							      >	      scheme: http
							      >	      kubernetes_sd_configs:
							      >	      - role: endpoints
							      >	        namespaces:
							      >	          names:
							      >	          - my-gpu-operator
							      >	      relabel_configs:
							      >	      - source_labels: [__meta_kubernetes_pod_node_name]
							      >	        action: replace
							      >	        target_label: kubernetes_node
                    

helm install prometheus-community/kube-prometheus-stack \
   --create-namespace --namespace prometheus \
   --generate-name \
   --values /tmp/kube-prometheus-stack.values.jerome

# Demo to be checked with MIG 
helm fetch https://helm.ngc.nvidia.com/nvidia/charts/video-analytics-demo-0.1.4.tgz && \
   helm install video-analytics-demo-0.1.4.tgz --generate-name
   
```
# Phase 6: Upgrade

First be sure you have pushed the  docker image version of the driver corresponding to the new operator.
So go back to Phase 1 and :
```
export DRIVER_VERSION=510.47.03
docker build -t driver:${DRIVER_VERSION}-sles15.3 --build-arg DRIVER_VERSION=${DRIVER_VERSION} .
```

Since Helm doesn’t support auto upgrade of existing CRDs, the user needs to follow a two step process to upgrade the GPU Operator chart.
It’s expected that during “helm upgrade”, helm doesn’t update the CRD to latest version automatically. Here are the steps to upgrade using helm:
- [nvidia GPU Operator Upgrade](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#upgrade)


```
# wget https://gitlab.com/nvidia/kubernetes/gpu-operator/-/raw/<release-tag>/deployments/gpu-operator/crds/nvidia.com_clusterpolicies_crd.yaml

# Step 1
wget https://gitlab.com/nvidia/kubernetes/gpu-operator/-/raw/v1.10.0/deployments/gpu-operator/crds/nvidia.com_clusterpolicies_crd.yaml
kubectl apply -f nvidia.com_clusterpolicies_crd.yaml

# Step 2
helm show values nvidia/gpu-operator --version=1.10.0 > values-1.10.0.yaml
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator -f values-1.10.0.yaml 

```

# Phase 7: uninstall
```
k delete -f vector-add/jear-vector-add1-1g.yaml && \
k delete -f vector-add/jear-vector-add2-1g.yaml && \
k delete -f dcgmproftester/dcgmproftester-mixed-2g.yaml && \
k delete -f tf-benchmarks/tf-benchmarks-mixed-3g.yaml

helm delete prometheus -n prometheus
helm delete  gpu-operator  -n my-gpu-operator 
```

- [nvidia GPU Operator Uninstall](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#uninstall)


