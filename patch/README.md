```
kubectl patch daemonset  -n gpu-operator nvidia-container-toolkit-daemonset  --patch "$(curl -sL https://gist.githubusercontent.com/jear/0c3f6577eee2f871712781813764e5f0/raw/8c6f7569458f80bce45e57a6a004a0b2fd7415a9/patch-rke2-nvidia-container-toolkit-daemonset.yaml)"
```
