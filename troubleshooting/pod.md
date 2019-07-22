# Pod Exception Troubleshooting

This chapter describes troubleshooting methods for Pod runtime exceptions.

In general, regardless of the abnormal state of the Pod, you can execute the following command to view the status of the Pod.

- `kubectl get pod <pod-name> -o yaml` Check if the Pod configuration is correct
- `kubectl describe pod <pod-name>` View the events of the Pod
- `kubectl logs <pod-name> [-c <container-name>]` View container logs

These events and logs often help to troubleshoot problems with Pods.

## Pod has been in the Pending state

Pending indicates that the Pod has not been scheduled to a Node. You can use the `kubectl describe pod <pod-name>` command to view the current Pod event and determine why there is no scheduling. Such as

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x6 over 27s)  default-scheduler  0/4 nodes are available: 2 Insufficient cpu.
```

Possible reasons include

- Insufficient resources, all Nodes in the cluster do not satisfy the CPU, memory, GPU or temporary storage space of the Pod request. The solution is to delete unused Pods in the cluster or add new Nodes.
- The HostPort port is already occupied. It is recommended to use Service to open the service port.

## Pod is always in Waiting or ContainerCreating state

First, check the current Pod event with the `kubectl describe pod <pod-name>` command.

```sh
$ kubectl -n kube-system describe pod nginx-pod
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              1m                default-scheduler  Successfully assigned nginx-pod to node1
  Normal   SuccessfulMountVolume  1m                kubelet, gpu13     MountVolume.SetUp succeeded for volume "config-volume"
  Normal   SuccessfulMountVolume  1m                kubelet, gpu13     MountVolume.SetUp succeeded for volume "coredns-token-sxdmc"
  Warning  FailedSync             2s (x4 over 46s)  kubelet, gpu13     Error syncing pod
  Normal   SandboxChanged         1s (x4 over 46s)  kubelet, gpu13     Pod sandbox changed, it will be killed and re-created.
```

It can be found that the Pod's Sandbox container does not start properly. The specific reason is to view the Kubelet log:

```sh
$ journalctl -u kubelet
...
Mar 14 04:22:04 node1 kubelet[29801]: E0314 04:22:04.649912   29801 cni.go:294] Error adding network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
Mar 14 04:22:04 node1 kubelet[29801]: E0314 04:22:04.649941   29801 cni.go:243] Error while adding to cni network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
Mar 14 04:22:04 node1 kubelet[29801]: W0314 04:22:04.891337   29801 cni.go:258] CNI failed to retrieve network namespace path: Cannot find network namespace for the terminated container "c4fd616cde0e7052c240173541b8543f746e75c17744872aa04fe06f52b5141c"
Mar 14 04:22:05 node1 kubelet[29801]: E0314 04:22:05.965801   29801 remote_runtime.go:91] RunPodSandbox from runtime service failed: rpc error: code = 2 desc = NetworkPlugin cni failed to set up pod "nginx-pod" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
```

It is found that the cni0 bridge is configured with an IP address of a different network segment, and the bridge is deleted (the network plug-in will be automatically re-created) to be repaired.

```sh
$ ip link set cni0 down
$ brctl delbr cni0
```

In addition to the above errors, there are other possible reasons

- Image pull failed, for example
  - Configured the wrong image
  - Kubelet cannot access the image (domestic environment access `gcr.io` requires special handling)
  - The key configuration of the private image is incorrect.
  - The image is too large and the pull timeout (the kubelet's `--image-pull-progress-deadline` and `--runtime-request-timeout` options can be adjusted appropriately)
- CNI network error, generally need to check the configuration of the CNI network plugin, such as
  - Unable to configure Pod network
  - Unable to assign IP address
- The container fails to start, you need to check if the correct image is packaged or if the correct container parameters are configured



## Pod is in ImagePullBackOff state

This is usually caused by a wrong configuration of the mirror name or a key configuration error for the private image. In this case, you can use `docker pull <image>` to verify that the image can be pulled normally.

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason                 Age                From                                Message
  ----     ------                 ----               ----                                -------
  Normal   Scheduled              36s                default-scheduler                   Successfully assigned sh to k8s-agentpool1-38622806-0
  Normal   SuccessfulMountVolume  35s                kubelet, k8s-agentpool1-38622806-0  MountVolume.SetUp succeeded for volume "default-token-n4pn6"
  Normal   Pulling                17s (x2 over 33s)  kubelet, k8s-agentpool1-38622806-0  pulling image "a1pine"
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Failed to pull image "a1pine": rpc error: code = Unknown desc = Error response from daemon: repository a1pine not found: does not exist or no pull access
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Error: ErrImagePull
  Normal   SandboxChanged         4s (x7 over 28s)   kubelet, k8s-agentpool1-38622806-0  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff                4s (x5 over 25s)   kubelet, k8s-agentpool1-38622806-0  Back-off pulling image "a1pine"
  Warning  Failed                 1s (x6 over 25s)   kubelet, k8s-agentpool1-38622806-0  Error: ImagePullBackOff
```

If it is a private image, you first need to create a Secret of type docker-registry

```sh
kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

Then reference this Secret in the container

`` Yaml
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: my-secret
```

## Pod is always in CrashLoopBackOff state

The CrashLoopBackOff state indicates that the container was started but exited abnormally. At this time, the RestartCounts of Pod is usually greater than 0. You can check the log of the container first.

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs --previous <pod-name>
```

Here you can find some reasons for the container to exit, such as

- Container process exits
- Health check failed to exit
- OOMKilled

```sh
$ kubectl describe pod mypod
...
Containers:
  sh:
    Container ID:  docker://3f7a2ee0e7e0e16c22090a25f9b6e42b5c06ec049405bc34d3aa183060eb4906
    Image:         alpine
    Image ID:      docker-pullable://alpine@sha256:7b848083f93822dd21b0a2f14a110bd99f6efb4b838d499df6d04a49d0debf8b
    Port:          <none>
    Host Port:     <none>
    State:          Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     1
      memory:  1G
    Requests:
      cpu:        100m
      memory:     500M
...
```

If you have not found a clue at this time, you can also execute the command in the container to further view the reason for the exit.

```sh
kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

If there is still no clue, then you need to SSH into the Node where the Pod is located, and check the Kubelet or Docker logs for further troubleshooting.

```sh
# Query Node
kubectl get pod <pod-name> -o wide

# SSH to Node
ssh <username>@<node-name>
```

## Pod is in Error state

Usually in the Error state indicates that an error occurred during Pod startup. Common reasons include

- DependMap, Secret or PV does not exist
- The requested resource exceeds the limit set by the administrator, such as exceeding LimitRange, etc.
- Violation of cluster security policies, such as violations of PodSecurityPolicy, etc.
- The container does not have permission to operate resources in the cluster. For example, after RBAC is enabled, you need to configure role binding for ServiceAccount.

## Pod is in Terminating or Unknown state

Starting with v1.5, Kubernetes does not delete a running Pod because Node is out of association, but marks it as a Terminating or Unknown state. There are three ways to delete a Pod in these states:

- Remove the Node from the cluster. When using a public cloud, kube-controller-manager automatically deletes the corresponding Node after the VM is deleted. In a cluster where physical machines are deployed, administrators need to manually delete Nodes (such as `kubectl delete node <node-name>`.
- Node is back to normal. Kubelet will re-communicate with kube-apiserver to confirm the expected state of these Pods, and then decide to delete or continue to run these Pods.
- User forced deletion. Users can execute `kubectl delete pods <pod> --grace-period=0 --force` to force the removal of Pods. This method is not recommended unless it is explicitly known that the Pod is indeed in a stopped state (such as the VM where the Node is located or the physical machine is powered off). In particular, Pod managed by StatefulSet, forced deletion can easily lead to brain splitting or data loss.

If Kubelet is running as a Docker container, the following error may be found in the kubelet log (https://github.com/kubernetes/kubernetes/issues/51835):

```json
{"log":"I0926 19:59:07.162477   54420 kubelet.go:1894] SyncLoop (DELETE, \"api\"): \"billcenter-737844550-26z3w_meipu(30f3ffec-a29f-11e7-b693-246e9607517c)\"\n","stream":"stderr","time":"2017-09-26T11:59:07.162748656Z"}
{"log":"I0926 19:59:39.977126   54420 reconciler.go:186] operationExecutor.UnmountVolume started for volume \"default-token-6tpnm\" (UniqueName: \"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\") pod \"30f3ffec-a29f-11e7-b693-246e9607517c\" (UID: \"30f3ffec-a29f-11e7-b693-246e9607517c\") \n","stream":"stderr","time":"2017-09-26T11:59:39.977438174Z"}
{"log":"E0926 19:59:39.977461   54420 nestedpendingoperations.go:262] Operation for \"\\\"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\\\" (\\\"30f3ffec-a29f-11e7-b693-246e9607517c\\\")\" failed. No retries permitted until 2017-09-26 19:59:41.977419403 +0800 CST (durationBeforeRetry 2s). Error: UnmountVolume.TearDown failed for volume \"default-token-6tpnm\" (UniqueName: \"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\") pod \"30f3ffec-a29f-11e7-b693-246e9607517c\" (UID: \"30f3ffec-a29f-11e7-b693-246e9607517c\") : remove /var/lib/kubelet/pods/30f3ffec-a29f-11e7-b693-246e9607517c/volumes/kubernetes.io~secret/default-token-6tpnm: device or resource busy\n","stream":"stderr","time":"2017-09-26T11:59:39.977728079Z"}
```

If this is the case, you need to set the `--containerized` parameter to the kubelet container and pass in the following storage volume.

```sh
# Using the calico network plugin as an example
      -v /:/rootfs:ro,shared \
      -v / sys: / sys: ro \
      -v /dev:/dev:rw \
      -v / var / log: / var / log: rw
      -v /run/calico/:/run/calico/:rw \
      -v /run/docker/:/run/docker/:rw \
      -v /run/docker.sock:/run/docker.sock:rw \
      -v /usr/lib/os-release:/etc/os-release \
      -v /usr/share/ca-certificates/:/etc/ssl/certs \
      -v /var/lib/docker/:/var/lib/docker:rw,shared \
      -v / var / lib / kubelet /: / var / lib / kubelet: rw, shared \
      -v / etc / kubernetes / ssl /: / etc / kubernetes / ssl / \
      -v /etc/kubernetes/config/:/etc/kubernetes/config/ \
      -v /etc/cni/net.d/:/etc/cni/net.d/ \
      -v / opt / cni / bin /: / opt / cni / bin / \
```

Pods in the `Terminating` state are usually automatically deleted after the Kubelet resumes normal operation. But sometimes it can't be deleted, and it can't be forced to delete by `kubectl delete pods <pod> --grace-period=0 --force`. This is usually caused by `finalizers`, which can be solved by deleting the finalizers with `kubectl edit`.

`` Yaml
"finalizers": [
  "foregroundDeletion"
]
```

## Pod behaves abnormally

The behavior exception mentioned here means that the Pod is not performing as expected, such as not running the command line parameters set in the podSpec. This is generally the content of the podSpec yaml file is incorrect, you can try to use the `--validate` parameter to rebuild the container, such as

```sh
kubectl delete pod mypod
kubectl create --validate -f mypod.yaml
```

You can also check if the created podSpec is correct, for example

```sh
kubectl get pod mypod -o yaml
```

## Modifying the static Pod's Manifest does not automatically rebuild

Kubelet uses the inotify mechanism to detect changes to the static Pod in the `/etc/kubernetes/manifests` directory (which can be specified by Kubelet's `--pod-manifest-path` option) and recreate the corresponding Pod after the file has changed. But sometimes it happens that the new Pod is not automatically created after modifying the Manifest of the static Pod. A simple fix is ​​to restart the Kubelet.

### Reference Document

- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)
