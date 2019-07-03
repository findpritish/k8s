# Kubernetes Storage Volume

We know that by default the container's data is non-persistent, and the data is lost after the container dies, so Docker provides a Volume mechanism to persist data. Similarly, Kubernetes provides a more powerful Volume mechanism and a rich set of plug-ins that solve the problem of container data persistence and shared data between containers.

Unlike Docker, Kubernetes Volume's lifecycle is tied to Pod

- When the container is hanged and Kubelet restarts the container again, the volume data is still
- The Volume will be cleaned up when the Pod is deleted. Whether the data is lost depends on the specific volume type. For example, the data of emptyDir will be lost, but the data of PV will not be lost.

## Volume type

few Volume types:

- emptyDir
- hostPath
- nfs
- secret
- persistentVolumeClaim
- local
- gitRepo

Note that not all of these volumes are persistent, such as emptyDir, secret, gitRepo, etc. These volumes disappear as the Pod dies.

## emptyDir

If the Pod is set to the emptyDir type Volume, the Pod is assigned to the Node, an emptyDir is created. As long as the Pod is running on the Node, the emptyDir will exist (the container will not cause the emptyDir to lose data), but if the Pod is deleted from the Node, (Pod is deleted, or Pod is migrated), emptyDir is also deleted and permanently lost.


``` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

## hostPath

hostPath allows you to mount the file system on Node to the Pod. If the Pod needs to use a file on Node, you can use hostPath.

``` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
```
