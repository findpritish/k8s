# ConfigMap

In the implementation of the application or production environment, there are many situations that need to be changed, and we do not want to prepare an image file for each type of demand, then we can help us make a configuration file through ConfigMap. Or the mapping of command parameters, more flexible use of our services or applications.

ConfigMap is used to save key-value pairs of configuration data. It can be used to save a single attribute or to save a configuration file. ConfigMap is similar to secret, but it makes it easier to work with strings that don't contain sensitive information.

## API version comparison table

| Kubernetes Version | Core API Version |
| --------------- | ------------- |
| v1.5+           | core/v1       |

## ConfigMap creation

You can create a ConfigMap from a file, directory, or key-value string creation using `kubectl create configmap`. It can also be created with `kubectl create -f file`.

### Created from a key-value string

```sh
$ kubectl create configmap special-config --from-literal=special.how=very
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[special.how:very]
```

### Created from an env file

```sh
$ echo -e "a=b\nc=d" | tee config.env
a=b
c=d
$ kubectl create configmap special-config --from-env-file=config.env
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[a:b c:d]
```

### Create from directory

```sh
$ mkdir config
$ echo a>config/a
$ echo b>config/b
$ kubectl create configmap special-config --from-file=config/
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[a:a
 b:b
]
```

### Created from a file Yaml/Json file

`` Yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

```sh
$ kubectl create  -f  config.yaml
configmap "special-config" created
```

## ConfigMap Use

ConfigMap can be used in Pod in three ways: by setting environment variables, setting container command line parameters, and mounting files or directories directly in the Volume.

> **Note**
>
> - ConfigMap must be created before Pod references it
> - Invalid keys are automatically ignored when using `envFrom`
> - Pod can only use ConfigMap in the same namespace

First create a ConfigMap:

```sh
$ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
$ kubectl create configmap env-config --from-literal=log_level=INFO
```

### Used as an environment variable

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "env"]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
            name: env-config
  restartPolicy: Never
```

Output when Pod ends

```
SPECIAL_LEVEL_KEY=very
SPECIAL_TYPE_KEY=charm
log_level=INFO
```

### Used as a command line argument

When using ConfigMap as a command line parameter, you need to save the ConfigMap data in the environment variable and then reference the environment variable by `$(VAR_NAME)`.

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```

Output when Pod ends

```
very charm
```

### Use volume to mount ConfigMap directly as a file or directory

The created ConfigMap is directly mounted to the Pod's /etc/config directory. Each key-value key-value pair will generate a file, the key is the file name, and the value is the content.

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "cat /etc/config/special.how"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

Output when Pod ends

```
very
```

Mount the special.how key in the created ConfigMap to a relative path / keys/special.level in the /etc/config directory. If there is a file with the same name, it will be overwritten directly. Other keys are not mounted

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh","-c","cat /etc/config/keys/special.level"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
  restartPolicy: Never
```
Output when Pod ends

```
very
```

ConfigMap supports mounting multiple keys and multiple directories in the same directory. For example, the special.how and special.type are mounted below under /etc/config. Also, special.how is also mounted to /etc/config2.

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh","-c","sleep 36000"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
      - name: config-volume2
        mountPath: /etc/config2
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
        - key: special.type
          path: keys/special.type
    - name: config-volume2
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
  restartPolicy: Never
```

```sh
# ls  /etc/config/keys/
special.level  special.type
# ls  /etc/config2/keys/
special.level
# cat  /etc/config/keys/special.level
very
# cat  /etc/config/keys/special.type
charm
```

### Mounting ConfigMap as a separate file to the directory using subpath
In general, when configmap mounts a file, it will overwrite the mount directory and then mount the contents of the congfigmap as a file. If you want to not overwrite the files in the original folder, just mount each key in the configmap to the directory according to the file. You can use the subpath parameter.
`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      command: ["/bin/sh","-c","sleep 36000"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/special.how
        subPath: special.how
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: special.how
  restartPolicy: Never
```

```sh
root@dapi-test-pod:/# ls /etc/nginx/
conf.d	fastcgi_params	koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params	special.how  uwsgi_params  win-utf
root@dapi-test-pod:/# cat /etc/nginx/special.how
very
root@dapi-test-pod:/#
```

Reference documentation:

* [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
