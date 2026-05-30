# Installation
```
helm repo add csi-s3 https://yandex-cloud.github.io/k8s-csi-s3/charts
helm install csi-s3 csi-s3/csi-s3 --namespace kube-system
kubectl delete storageclass csi-s3
kubectl apply -k k8s/base/
```