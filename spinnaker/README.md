## Spinnaker deployment to Kubernetes

Start Minikube

`$ minikube start --memory 4096 --cpus 4`

`$ minikube config set cpus 4`

`$ kubectl version`

`$ kubectl get cs`

Apply the below yaml to your cluster 

`$ kubectl apply -f https://github.com/findpritish/k8s/new/master/spinnaker/deploy-spinnaker.yaml`

Before we access the Spinnaker dashboard from the browser, we need to enable port forwarding by running the below commands. This exposes the Pod that runs the Spinnaker web UI to the host.

`$ export DECK_POD=$(kubectl get pods --namespace spinnaker -l "component=deck,app=kubelive-spinnaker" -o jsonpath="{.items[0].metadata.name}")

`$ kubectl port-forward --namespace spinnaker $DECK_POD 9000`

Spinnaker can now be accessed from the browser by visiting http://localhost:9000.
