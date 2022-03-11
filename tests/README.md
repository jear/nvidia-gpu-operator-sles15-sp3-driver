# Test 1: change MIG strategy and disabling MIG mode
```
# Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed --wait

# Single MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=single --wait
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "single",


# Back in Mixed MIG strategy
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed --wait

# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  

```


# Test 2: While in Single mode, try to apply a mixed geometry -> should fail

Examples here : https://github.com/NVIDIA/mig-parted/blob/master/examples/config.yaml

```
# Choose your geometry
k describe configmaps -n my-gpu-operator default-mig-parted-config

# Apply your geometry
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

--> fail -> OK

# back to a consistent geometry/strategy
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-1g.5gb --overwrite  

# in single mode
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "7",




# in mixed mode
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "1",
  "nvidia.com/mig-1g.5gb.count": "7",



```

# Test 3 : Workload Test in mixed mode (all-balanced 1g, 2g, 3g)

```
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

k apply -f jear-vector-add1-1g.yaml && \
k apply -f jear-vector-add2-1g.yaml && \
k apply -f dcgmproftester-mixed-2g.yaml && \
k apply -f tf-benchmarks-mixed-3g.yaml

```

