# Phase1: Build Driver (don't do that if NVIDIA publish the offical driver
```
zypper in docker
systemctl enable docker --now
git clone https://gitlab.com/fcrozat/nvidia-driver
cd nvidia-driver/sle15
export DRIVER_VERSION=470.82.01
docker build -t driver:${DRIVER_VERSION}-sles15.3 --build-args DRIVER_VERSION=${DRIVER_VERSION} .

```
# Phase 2: Deploy gpu-operator
```
helm upgrade --install gpu-operator  nvidia/gpu-operator  -n my-gpu-operator --create-namespace  --set mig.strategy=mixed
```

# Phase 3: Configure geometry
```
# Choose your geometry

# Apply your geometry
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-balanced --overwrite  

# Disable MIG mode and get 1 GPU
kubectl label nodes worker-gpu-7 nvidia.com/mig.config=all-disabled --overwrite  


```
# Phase 4 : Profit

```

```
