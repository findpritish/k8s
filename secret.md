# Secret

Secret solves the configuration problem of sensitive data such as passwords, tokens, keys, etc., without exposing these sensitive data to images or Pod Spec. Secret can be used as a Volume or as an environment variable.

## Secret Type

There are three types of Secret:
* Opaque: The secret of the base64 encoding format, used to store passwords, keys, etc.; but the data is also decoded by base64 --decode to get the original data, all encryption is very weak.
* `kubernetes.io/dockerconfigjson`: Used to store authentication information for the private docker registry.
* `kubernetes.io/service-account-token`: Used to be referenced by serviceaccount. When serviceaccout is created, Kubernetes will create the corresponding secret by default. Pod If serviceaccount is used, the corresponding secret will be automatically mounted to the Pod's `/run/secrets/kubernetes.io/serviceaccount` directory.

Note: serviceaccount is used to enable Pod to access the Kubernetes API

## API version comparison table

| Kubernetes Version | Core API Version |
| --------------- | ------------- |
| v1.5+           | core/v1       |

## Opaque Secret

The Opaque type data is a map type, and the value is required to be in base64 encoding format:

```sh
$ echo -n "admin" | base64
YWRtaW4 =
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

secrets.yml

`` Come
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

Create secret:`kubectl create -f secrets.yml`.
```sh
# kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-cty7p   kubernetes.io/service-account-token   3         45d
mysecret              Opaque                                2         7s
```
Note: where default-token-cty7p is the default secret created when the cluster is created, and is referenced by serviceacount/default.

If you create a secret from a file, you can use a simpler kubectl command, such as creating a secret for tls:

```sh
$ kubectl create secret generic helloworld-tls \
  --from-file=key.pem \
  --from-file=cert.pem
```

## Use of Opaque Secret

After creating a secret, there are two ways to use it:

* in Volume mode
* in the form of environmental variables

### Mount Secret to Volume

`` Come
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
  containers:
  - image: gcr.io/my_project_id/pg:v1
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 5432
      hostPort: 5432
```

View the corresponding information in the Pod:
```sh
# ls /etc/secrets
password  username
# cat  /etc/secrets/username
admin
# cat  /etc/secrets/password
1f2d1e2e67df
```

### Export Secret to environment variables


`` Come
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  replicas: 2
  strategy:
      type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress
        visualize: "true"
    spec:
      containers:
      - name: "wordpress"
        image: "wordpress"
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

### Mount Secret with the specified key
`` Come
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
      items:
      - key: password
        mode: 511
        path: tst/psd
      - key: username
        mode: 511
        path: tst/usr
  containers:
  - image: nginx
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 80
      hostPort: 5432
```
After creating a Pod successfully, you can see it in the corresponding directory:
```sh
# kubectl exec db ls /etc/secrets/tst
psd
usr
```

** Note **:

1. The secrets of the `kubernetes.io/dockerconfigjson` and `kubernetes.io/service-account-token` types can also be mounted as files (directories).
If you use the secret of type `kubernetes.io/dockerconfigjson`, a . dockercfg file will be created in the directory.
```sh
root@db:/etc/secrets# ls -al
total 4
drwxrwxrwt  3 root root  100 Aug  5 16:06 .
drwxr-xr-x 42 root root 4096 Aug  5 16:06 ..
drwxr-xr-x  2 root root   60 Aug  5 16:06 ..8988_06_08_00_06_52.433429084
lrwxrwxrwx  1 root root   31 Aug  5 16:06 ..data -> ..8988_06_08_00_06_52.433429084
lrwxrwxrwx  1 root root   17 Aug  5 16:06 .dockercfg -> ..data/.dockercfg
```
If you use the secret of type `kubernetes.io/service-account-token`, you will create three files ca.crt, namespace, and token.
```sh
root@db:/etc/secrets# ls
ca.crt	namespace  token
```
2. The secrets are mounted to a temporary directory when they are used. The files generated when the secrets are mounted after the Pod is deleted are also deleted.
```sh
root@db:/etc/secrets# df
Filesystem     1K-blocks    Used Available Use% Mounted on
none           123723748 4983104 112432804   5% /
tmpfs            1957660       0   1957660   0% /dev
tmpfs            1957660       0   1957660   0% /sys/fs/cgroup
/dev/vda1       51474044 2444568  46408092   6% /etc/hosts
tmpfs            1957660      12   1957648   1% /etc/secrets
/dev/vdb       123723748 4983104 112432804   5% /etc/hostname
shm 65536 0 65536 0% / dev / shm
```
But if the Pod is running, you can still see it on the node where Pod is deployed:
```sh
# View information about the container Secret in the Pod, where 4392b02d-79f9-11e7-a70a-525400bc11f0 is the UUID of the Pod
"Mounts": [
  {
    "Source": "/var/lib/kubelet/pods/4392b02d-79f9-11e7-a70a-525400bc11f0/volumes/kubernetes.io~secret/secrets",
    "Destination": "/etc/secrets",
    "Mode": "ro",
    "RW": false,
    "Propagation": "rprivate"
  }
]
#View on the node deployed by Pod
root@VM-0-178-ubuntu:/var/lib/kubelet/pods/4392b02d-79f9-11e7-a70a-525400bc11f0/volumes/kubernetes.io~secret/secrets# ls -al
total 4
drwxrwxrwt 3 root root  140 Aug  6 00:15 .
drwxr-xr-x 3 root root 4096 Aug  6 00:15 ..
drwxr-xr-x 2 root root  100 Aug  6 00:15 ..8988_06_08_00_15_14.253276142
lrwxrwxrwx 1 root root   31 Aug  6 00:15 ..data -> ..8988_06_08_00_15_14.253276142
lrwxrwxrwx 1 root root   13 Aug  6 00:15 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root   16 Aug  6 00:15 namespace -> ..data/namespace
lrwxrwxrwx 1 root root   12 Aug  6 00:15 token -> ..data/token
```

## kubernetes.io/dockerconfigjson

You can use the kubectl command to create a secret for docker registry authentication:

```sh
$ kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistrykey" created.
```
View the contents of secret:
```sh
# get secret myregistrykey -o yaml
apiVersion: v1
data:
  .dockercfg: eyJjY3IuY2NzLnRlbmNlbnR5dW4uY29tL3RlbmNlbnR5dW4iOnsidXNlcm5hbWUiOiIzMzIxMzM3OTk0IiwicGFzc3dvcmQiOiIxMjM0NTYuY29tIiwiZW1haWwiOiIzMzIxMzM3OTk0QHFxLmNvbSIsImF1dGgiOiJNek15TVRNek56azVORG94TWpNME5UWXVZMjl0In19
kind: Secret
metadata:
  creationTimestamp: 2017-08-04T02:06:05Z
  name: myregistrykey
  namespace: default
  resourceVersion: "1374279324"
  selfLink: / api / v1 / namespaces / default / secrets / myregistrykey
  uid: 78f6a423-78b9-11e7-a70a-525400bc11f0
type: kubernetes.io/dockercfg
```

Decode the contents of the secret via base64:
```sh
# echo "eyJjY3IuY2NzLnRlbmNlbnR5dW4uY29tL3RlbmNlbnR5dW4iOnsidXNlcm5hbWUiOiIzMzIxMzM3OTk0IiwicGFzc3dvcmQiOiIxMjM0NTYuY29tIiwiZW1haWwiOiIzMzIxMzM3OTk0QHFxLmNvbSIsImF1dGgiOiJNek15TVRNek56azVORG94TWpNME5UWXVZMjl0XXXX" | base64 --decode
{"ccr.ccs.tencentyun.com/XXXXXXX":{"username":"3321337XXX","password":"123456.com","email":"3321337XXX@qq.com","auth":"MzMyMTMzNzk5NDoxMjM0NTYuY29t"}}
```

You can also directly read the contents of `~/.dockercfg` to create:

```sh
$ kubectl create secret docker-registry myregistrykey \
  --from-file="~/.dockercfg"
```

When creating a Pod, refer to the `myregistrykey` you just created by `imagePullSecrets`:

`` Yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```

### kubernetes.io/service-account-token

`kubernetes.io/service-account-token`: Used to be referenced by serviceaccount. When serviceaccout is created, Kubernetes will create the corresponding secret by default. Pod If serviceaccount is used, the corresponding secret will be automatically mounted to the Pod's `/run/secrets/kubernetes.io/serviceaccount` directory.

```sh
$ kubectl run nginx --image nginx
deployment "nginx" created
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-3137573019-md1u2   1/1       Running   0          13s
$ kubectl exec nginx-3137573019-md1u2 ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

## Storage Encryption

The v1.7+ version supports encrypting Secret data to etcd, and only needs to configure `--experimental-encryption-provider-config` when apiserver starts. The encryption configuration format is

`` Yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
    - identity: {}
    - aesgcm;
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
    - secretbox:
        keys:
        - name: key1
          secret: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
```

among them

- resources.resources is the resource name of Kubernetes
- resources.providers is an encryption method that supports the following
  - identity: no encryption
  - aescbc: AES-CBC encryption
  - secretbox: XSalsa20 and Poly1305 encryption
  - aesgcm: AES-GCM encryption

Secret is encrypted when writing storage, so you can perform an update operation on the existing secret to ensure that all secrets are encrypted.

```sh
kubectl get secrets -o json | kubectl update -f -
```

If you want to cancel the secret encryption, just put `identity` in the first place of the provider (aeesbcc is still needed to access the stored secret):

`` Yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
    - secrets
    providers:
    - identity: {}
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
```

## Secret vs. ConfigMap

Same point:
- the form of key/value
- belong to a specific namespace
- Can be exported to environment variables
- Can be mounted by directory/file (supports mounting all keys and partial keys)

difference:
- Secret can be associated (used) by ServerAccount
- Secret can store the register's authentication information, used in the ImagePullSecret parameter, to pull the image of the private repository
- Secret supports Base64 encryption
- Secret is divided into Opaque, kubernetes.io/Service Account, kubernetes.io/dockerconfigjson three types, Configmap does not distinguish between types
- The Secret file is stored in the tmpfs file system. After the Pod is deleted, the Secret file is also deleted.


## Reference Document

- [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Specifying ImagePullSecrets on a Pod](https://kubernetes.io/docs/concepts/configuration/secret/)
