# Setup Datadog for kubernetes Cluster 

In this article I’m going to show you how to deploy the DataDog agent to your Kubernetes cluster as a DaemonSet

First we need to make a Kubernetes Secret so that we dont have the API key in plaintext in our datadog-ds.yml.

Kubernetes secrets use the base64 encoded value. So lets get that


```bash

$ echo -n "my_datadog_api_key" | base64

$ kubectl create -f datadog-api-key-secret.yml
# do not commit datadog-api-key-secret.yml to sourcecontrol with secret key

```

```bash

$ kubectl create -f clusterrole.yml
$ kubectl create -f serviceaccount.yml
$ kubectl create -f clusterrolebinding.yml

```
Apply daemonset to your cluster

```bash
$ kubectl create -f datadog-ds.yml

```

Happy metrix flow from cluster to DG
