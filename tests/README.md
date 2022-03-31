# Test 1: change MIG strategy 
If you just want to change the “mig.strategy” without intention to upgrade operator, you can always edit clusterpolicy to change it using “kubectl edit clusterpolicy”, operator will apply that change.
```
kubectl edit clusterpolicy
...
  mig:
    strategy: mixed
...
```


# Test 2: disabling MIG mode and back in MIG enabled ( single or mixed )

```
# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "1",

k exec -n my-gpu-operator nvidia-mig-manager-kjm7h -- nvidia-smi
Defaulted container "nvidia-mig-manager" out of: nvidia-mig-manager, toolkit-validation (init)
Fri Mar 11 12:53:49 2022       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.82.01    Driver Version: 470.82.01    CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100-PCI...  On   | 00000000:61:00.0 Off |                    0 |
| N/A   66C    P0    44W / 250W |      0MiB / 40536MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+


# Back in Mixed MIG strategy
# check what strategy is already defined
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "mixed",

kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  
node/worker-gpu-7 labeled

kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count
  "nvidia.com/gpu.count": "1",
  "nvidia.com/mig-1g.5gb.count": "2",
  "nvidia.com/mig-2g.10gb.count": "1",
  "nvidia.com/mig-3g.20gb.count": "1",


```


# Test 3: While in Single mode, try to apply a mixed geometry -> should fail

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
kubectl edit clusterpolicy
...
  mig:
    strategy: mixed
...

kubectl get node -o json | jq '.items[].metadata.labels' | grep -i strategy
  "nvidia.com/mig.strategy": "mixed",

# in mixed strategy, try to apply a mixed-geometry and single strategy
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-1g.5gb --overwrite  


kubectl get node -o json | jq '.items[].metadata.labels' | grep -i count


-->  OK


```

# Test 4 : Workload Tests in mixed mode (all-balanced 1g, 2g, 3g)

```
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

k apply -f vector-add/jear-vector-add1-1g.yaml && \
k apply -f vector-add/jear-vector-add2-1g.yaml && \
k apply -f dcgmproftester/dcgmproftester-mixed-2g.yaml && \
k apply -f tf-benchmarks/tf-benchmarks-mixed-3g.yaml

```

