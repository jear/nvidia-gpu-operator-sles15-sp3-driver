# nvidia-gpu-operator-sles15-sp3-driver

Install
```
zypper in docker
systemctl enable docker --now
git clone https://gitlab.com/fcrozat/nvidia-driver
cd nvidia-driver/sle15
export DRIVER_VERSION=470.82.01
docker build -t driver:${DRIVER_VERSION}-sles15.3 --build-args DRIVER_VERSION=${DRIVER_VERSION} .
```
