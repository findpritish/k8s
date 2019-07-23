## Spark on Kubernetes Deployment

The Kubernetes example [github] (https://github.com/kubernetes/examples/tree/master/staging/spark) provides a detailed method of deploying the spark. Due to the complexity of the steps, here are some parts that are simplified so that everyone can install it. Go and set more things.

### Deployment conditions

* A kubernetes cluster
* kube-dns is working properly

### Create a namespace

namespace-spark-cluster.yaml

`` Yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "spark-cluster"
  labels:
    name: "spark-cluster"
```

```sh
$ kubectl create -f examples/staging/spark/namespace-spark-cluster.yaml
```

The original text mentioned here needs to transfer the execution environment of kubectl to spark-cluster. In order to facilitate us not to do this, we will add the following deployment namespace to spark-cluster.


### Deploying Master Service

Create a replication controller to run the Spark Master service

`` Yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-master-controller
  namespace: spark-cluster
spec:
  replicas: 1
  selector:
    component: spark-master
  template:
    metadata:
      labels:
        component: spark-master
    spec:
      containers:
        - name: spark-master
          image: gcr.io/google_containers/spark:1.5.2_v1
          command: ["/start-master"]
          ports:
            - containerPort: 7077
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
```


```sh
$ kubectl create -f spark-master-controller.yaml
```

Create a master service


spark-master-service.yaml

`` Yaml
kind: Service
apiVersion: v1
metadata:
  name: spark-master
  namespace: spark-cluster
spec:
  ports:
    - port: 7077
      targetPort: 7077
      name: spark
    - port: 8080
      targetPort: 8080
      name: http
  selector:
    component: spark-master
```

```sh
$ kubectl create -f spark-master-service.yaml
```

Check if the Master is working properly

```sh
$ kubectl get pod -n spark-cluster
spark-master-controller-qtwm8     1/1       Running   0          6d
```

```sh
$ kubectl logs spark-master-controller-qtwm8 -n spark-cluster
17/08/07 02:34:54 INFO Master: Registered signal handlers for [TERM, HUP, INT]
17/08/07 02:34:54 INFO SecurityManager: Changing view acls to: root
17/08/07 02:34:54 INFO SecurityManager: Changing modify acls to: root
17/08/07 02:34:54 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(root); users with modify permissions: Set(root)
17/08/07 02:34:55 INFO Slf4jLogger: Slf4jLogger started
17/08/07 02:34:55 INFO Remoting: Starting remoting
17/08/07 02:34:55 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkMaster@spark-master:7077]
17/08/07 02:34:55 INFO Utils: Successfully started service 'sparkMaster' on port 7077.
17/08/07 02:34:55 INFO Master: Starting Spark master at spark://spark-master:7077
17/08/07 02:34:55 INFO Master: Running Spark version 1.5.2
17/08/07 02:34:56 INFO Utils: Successfully started service 'MasterUI' on port 8080.
17/08/07 02:34:56 INFO MasterWebUI: Started MasterWebUI at http://10.2.6.12:8080
17/08/07 02:34:56 INFO Utils: Successfully started service on port 6066.
17/08/07 02:34:56 INFO StandaloneRestServer: Started REST server for submitting applications on port 6066
17/08/07 02:34:56 INFO Master: I have been elected leader! New state: ALIVE
```


If the master has been set up and running, we can view the cluster status of our spark through the webUI developed by Spark. We will deploy [specialized proxy] (https://github.com/aseigneurin/spark-ui-proxy)


spark-ui-proxy-controller.yaml

`` Yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-ui-proxy-controller
  namespace: spark-cluster
spec:
  replicas: 1
  selector:
    component: spark-ui-proxy
  template:
    metadata:
      labels:
        component: spark-ui-proxy
    spec:
      containers:
        - name: spark-ui-proxy
          image: elsonrodriguez/spark-ui-proxy:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
          args:
            - spark-master:8080
          livenessProbe:
              httpGet:
                path: /
                port: 80
              initialDelaySeconds: 120
              timeoutSeconds: 5
```

```sh
$ kubectl create -f spark-ui-proxy-controller.yaml
```

Provide a service to access, here the original is to use LoadBalancer type, here we changed to NodePort, if your kubernetes running environment is in the cloud provider, you can also refer to the original text

spark-ui-proxy-service.yaml

`` Yaml
kind: Service
apiVersion: v1
metadata:
  name: spark-ui-proxy
  namespace: spark-cluster
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  selector:
    component: spark-ui-proxy
  type: NodePort
```

```sh
$ kubectl create -f spark-ui-proxy-service.yaml
```

After deployment, you can use [kubecrl proxy] (https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/) to view your Spark cluster status.

```sh
$ kubectl proxy --port=8001
```

Can be viewed through `http://localhost:8001/api/v1/proxy/namespaces/spark-cluster/services/spark-master:8080/`, if kubectl is interrupted, it can't be observed like this, but we have previously set it up. The nodeport can also be viewed through the port 30080 of any node (for example, `http://10.201.2.34:30080`).

### Deploying Spark workers

First make sure the Matser is running again.

spark-worker-controller.yaml
```
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-worker-controller
  namespace: spark-cluster
spec:
  replicas: 2
  selector:
    component: spark-worker
  template:
    metadata:
      labels:
        component: spark-worker
    spec:
      containers:
        - name: spark-worker
          image: gcr.io/google_containers/spark:1.5.2_v1
          command: ["/start-worker"]
          ports:
            - containerPort: 8081
          resources:
            requests:
              cpu: 100m
```

```sh
$ kubectl create -f spark-worker-controller.yaml
replicationcontroller "spark-worker-controller" created
```

Check the operating status through instructions

```sh
$ kubectl get pod -n spark-cluster
spark-master-controller-qtwm8     1/1       Running   0          6d
spark-worker-controller-4rxrs     1/1       Running   0          6d
spark-worker-controller-z6f21     1/1       Running   0          6d
spark-ui-proxy-controller-d4br2   1/1       Running   4          6d
```

You can also view it through the WebUI service created above.

Basically, the cluster of Spark has been established here.


### Create Zeppelin UI

We can use Zeppelin UI to perform our tasks directly via web notebook.
See [Zeppelin UI] (http://zeppelin.apache.org/) and [Spark architecture] for more details (https://spark.apache.org/docs/latest/cluster-overview.html)

zeppelin-controller.yaml

`` Yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: zeppelin-controller
  namespace: spark-cluster
spec:
  replicas: 1
  selector:
    component: zeppelin
  template:
    metadata:
      labels:
        component: zeppelin
    spec:
      containers:
        - name: zeppelin
          image: gcr.io/google_containers/zeppelin:v0.5.6_v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
```


```sh
$ kubectl create -f zeppelin-controller.yaml
replicationcontroller "zeppelin-controller" created
```

Then deploy the Service

zeppelin-service.yaml

```sh
kind: Service
apiVersion: v1
metadata:
  name: zeppelin
  namespace: spark-cluster
spec:
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30081
  selector:
    component: zeppelin
  type: NodePort
```

```sh
$ kubectl create -f zeppelin-service.yaml
```

You can see that we have set the NodePort to 30081, and the zeppelin UI can be accessed through the 30081 port of any node.

Access pyspark from the command line (remember to replace the pod name with your own):

```
$ kubectl exec -it zeppelin-controller-8f14f -n spark-cluster pyspark
Python 2.7.9 (default, Mar  1 2015, 12:57:24)
[GCC 4.9.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
17/08/14 01:59:22 WARN Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 1.5.2
      /_/

Using Python version 2.7.9 (default, Mar  1 2015 12:57:24)
SparkContext available as sc, HiveContext available as sqlContext.
>>>
```

You can then use Spark's services and welcome corrections if you have errors.

### zeppelin FAQ

* zeppelin's image is very large, so it will take some time to pull, and the size problem is now being solved. For details, please refer to issue #17231
* On the GKE platform, `kubectl post-forward` may be somewhat unstable. If you see the status of zeppelin is `Disconnected`, `port-forward` may have failed. You need to restart it. For details, please refer to #12179

## Reference Document

- [Apache Spark on Kubernetes](https://apache-spark-on-k8s.github.io/userdocs/index.html)
- [https://github.com/kweisamx/spark-on-kubernetes](https://github.com/kweisamx/spark-on-kubernetes)
- [Spark examples](https://github.com/kubernetes/examples/tree/master/staging/spark)
