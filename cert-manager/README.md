
# create certmanager and its related ingress (applicable only for selfsigned cert with letsencrypt)

# create namespace for certmanager
```
kubectl create namespace cert-manager
```

# create CRD and deployments for cert-manager
```
kubectl create -f cert-manager-01.yaml

kubectl create -f cert-manager-02.yaml

kubectl create -f cert-manager-03.yaml
```

# create ingress
```
kubectl create -f cert-manager-ingress-04.yaml
```
