# StatefulSet

StatefulSet is designed to solve the problem of stateful services (the corresponding Deployments and ReplicaSets are designed for stateless services), and its application scenarios include

- Stable persistent storage, that is, Pod can still access the same persistent data after rescheduling, based on PVC
- A stable network flag, that is, the PodName and HostName are unchanged after the Pod is rescheduled, based on the Headless Service (that is, the Service without Cluster IP).
- Ordered deployment, ordered extension, that is, Pod is sequential, in the order of definition or deployment in order of deployment (ie from 0 to N-1, all previous Pods must be before the next Pod runs) Both Running and Ready states), based on init containers
- Ordered contraction, ordered deletion (ie from N-1 to 0)

From the above application scenario, we can find that the StatefulSet consists of the following parts:

- Headless Service for defining a DNS domain
- volumeClaimTemplates for creating PersistentVolumes
- Define a StatefulSet for a specific application

The DNS format of each Pod in the StatefulSet is `statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`, where

- `serviceName` is the name of Headless Service
- `0..N-1` is the serial number of the Pod, starting from 0 to N-1
- `statefulSetName` is the name of the StatefulSet
- `namespace` is the namespace in which the service is located. Headless Service and StatefulSet must be in the same namespace.
- `.cluster.local` 为 Cluster Domain

## API version comparison table

| Kubernetes Version | Apps Version |
| ------------- | ------------------ |
|   v1.6-v1.7   | apps/v1beta1 |
|     v1.8      |   apps/v1beta2     |
|     v1.9      |      apps/v1       |

## Simple example

Take a simple nginx service [web.yaml] (web.txt) as an example:

`` Yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

```sh
$ kubectl create -f web.yaml
service "nginx" created
statefulset "web" created

# View the created headless service and statefulset
$ kubectl get service nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     None         <none>        80/TCP    1m
$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web 2 2 2 m

# Automatically create PVCs according to volumeClaimTemplates (volumes of type kubernetes.io/gce-pd are automatically created in GCE)
$ kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-d064a004-d8d4-11e6-b521-42010a800002   1Gi        RWO           16s
www-web-1   Bound     pvc-d06a3946-d8d4-11e6-b521-42010a800002   1Gi        RWO           16s

# View the created Pods, they are all ordered
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          5m
web-1     1/1       Running   0          4m

# Use nslookup to view the DNS of these Pods
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
/ # nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.2.10
/ # nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.3.12
/ # nslookup web-0.nginx.default.svc.cluster.local
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx.default.svc.cluster.local
Address 1: 10.244.2.10
```

Can also perform other operations

```sh
#扩容
$ kubectl scale statefulset web --replicas=5

#缩容
$ kubectl patch statefulset web -p '{"spec":{"replicas":3}}'

# Mirror update (currently does not support direct update of image, need patch to indirectly)
$ kubectl patch statefulset web --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"gcr.io/google_containers/nginx-slim:0.7"}]'

# Remove StatefulSet and Headless Service
$ kubectl delete statefulset web
$ kubectl delete service nginx

# StatefulSet After the deletion, the PVC will remain, and the data will need to be deleted if it is no longer used.
$ kubectl delete pvc www-web-0 www-web-1
```

## Update StatefulSet

V1.7 + supports automatic updating of StatefulSets, setting update policies via `spec.updateStrategy`. Currently supports two strategies

- OnDelete: When the `.spec.template` is updated, the old Pod is not deleted immediately, but the user is automatically deleted after the old Pod is manually deleted. This is the default update policy and is compatible with the v1.6 version of the behavior.
- RollingUpdate: When `.spec.template` is updated, the old Pod is automatically deleted and a new Pod replacement is created. At the time of the update, these Pods are performed in reverse order, sequentially deleting, creating, and waiting for the Pod to become Ready for the next Pod update.

### Partitions

RollingUpdate also supports Partitions, set by `.spec.updateStrategy.rollingUpdate.partition`. When the partition is set, only the Pod whose sequence number is greater than or equal to the partition will be updated when the `.spec.template` is updated, and the rest of the Pod will remain unchanged (even if it is deleted, it will be recreated with the previous version).

```sh
# Set partition to 3
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
statefulset "web" patched

# Update StatefulSet
$ kubectl patch statefulset web --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"gcr.io/google_containers/nginx-slim:0.7"}]'
statefulset "web" patched

#验证更新
$ kubectl delete by web-2
pod "web-2" deleted
$ kubectl get po -lapp = nginx -w
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          4m
web-1     1/1       Running             0          4m
web-2     0/1       ContainerCreating   0          11s
web-2     1/1       Running             0          18s
```

## Pod Management Strategy

V1.7 + can set Pod management policy through `.spec.podManagementPolicy`, support two ways

- OrderedReady: The default policy, creating each Pod in order of Pod and waiting for Ready before creating the following Pod
- Parallel: Create or delete Pods in parallel (start creating all Pods without waiting for the previous Pod Ready)

### Parallel example

`` Yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  podManagementPolicy: "Parallel"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

As you can see, all Pods are created in parallel.

```sh
$ kubectl create -f webp.yaml
service "nginx" created
statefulset "web" created

$ kubectl get po -lapp = nginx -w
NAME      READY     STATUS              RESTARTS  AGE
web-0     0/1       Pending             0         0s
web-0     0/1       Pending             0         0s
web-1     0/1       Pending             0         0s
web-1     0/1       Pending             0         0s
web-0     0/1       ContainerCreating   0         0s
web-1     0/1       ContainerCreating   0         0s
web-0     1/1       Running             0         10s
web-1     1/1       Running             0         10s
```

## zookeeper

Another example that better illustrates the power of StatefulSet is [zookeeper.yaml](zookeeper.txt).

`` Yaml
---
apiVersion: v1
kind: Service
metadata:
  name: zk-headless
  labels:
    app: zk-headless
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: zk-config
data:
  ensemble: "zk-0;zk-1;zk-2"
  jvm.heap: "2G"
  tick: "2000"
  init: "10"
  sync: "5"
  client.cnxns: "60"
  snap.retain: "3"
  purge.interval: "1"
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-budget
spec:
  selector:
    matchLabels:
      app: zk
  minAvailable: 2
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: zk-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: zk
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
        scheduler.alpha.kubernetes.io/affinity: >
            {
              "podAntiAffinity": {
                "requiredDuringSchedulingRequiredDuringExecution": [{
                  "labelSelector": {
                    "matchExpressions": [{
                      "key": "app",
                      "operator": "In",
                      "values": ["zk-headless"]
                    }]
                  },
                  "topologyKey": "kubernetes.io/hostname"
                }]
              }
            }
    spec:
      containers:
      - name: k8szk
        imagePullPolicy: Always
        image: gcr.io/google_samples/k8szk:v1
        resources:
          requests:
            memory: "4Gi"
            cpu: "1"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        env:
        - name: ZK_ENSEMBLE
          valueFrom:
            configMapKeyRef:
              name: zk-config
              key: together
        - name : ZK_HEAP_SIZE
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: jvm.heap
        - name : ZK_TICK_TIME
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name: ZK_INIT_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: init
        - name : ZK_SYNC_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name : ZK_MAX_CLIENT_CNXNS
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: client.cnxns
        - name: ZK_SNAP_RETAIN_COUNT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: snap.retain
        - name: ZK_PURGE_INTERVAL
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: purge.interval
        - name: ZK_CLIENT_PORT
          value: "2181"
        - name: ZK_SERVER_PORT
          value: "2888"
        - name: ZK_ELECTION_PORT
          value: "3888"
        command:
        - sh
        - -c
        - zkGenConfig.sh && zkServer.sh start-foreground
        readinessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 15
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

```sh
kubectl create -f zookeeper.yaml
```

For detailed instructions, see [zookeeper stateful application] (https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/).

## StatefulSet Notes

1. Recommended for use in Kubernetes v1.9 or later
2. All Pod's Volume must be created in advance using PersistentVolume or by the administrator.
3. To ensure data security, the Volume is not deleted when the StatefulSet is deleted.
4. StatefulSet requires a Headless Service to define the DNS domain, which needs to be created before the StatefulSet
