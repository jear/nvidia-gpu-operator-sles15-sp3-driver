# Test 1: change MIG strategy and disabling MIG mode
```
# Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed --wait
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "mixed",

# Single MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=single --wait
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "single",


# Back in Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed --wait
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "mixed",

# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  

```


# Test 2: While in Single mode, try to apply a mixed geometry -> should fail

Examples here : https://github.com/NVIDIA/mig-parted/blob/master/examples/config.yaml

```
# Choose your geometry
k describe configmaps -n my-gpu-operator default-mig-parted-config

# in single strategy, try to apply a mixed-geometry
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

--> fail -> OK

# back to a consistent geometry/strategy
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-1g.5gb --overwrite  

# in single mode
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "7",


# Back in Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed --wait
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "mixed",

# in mixed strategy, try to apply a mixed-geometry and single strategy
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-1g.5gb --overwrite  


kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count


-->  OK


```

# Test 3 : Workload Test in mixed mode (all-balanced 1g, 2g, 3g)

```
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

k apply -f jear-vector-add1-1g.yaml && \
k apply -f jear-vector-add2-1g.yaml && \
k apply -f dcgmproftester-mixed-2g.yaml && \
k apply -f tf-benchmarks-mixed-3g.yaml

```

