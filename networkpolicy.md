# Network Policy

With the popularity of microservices, more and more cloud service platforms require a large number of network calls between modules. Kubernetes introduced Network Policy in 1.3, which provides policy-based network control to isolate applications and reduce attack surfaces. It uses a tag selector to emulate a traditional segmentation network and controls the traffic between them and the traffic from the outside through policies.

Need to pay attention when using Network Policy

- v1.6 and previous versions need to open `extensions/v1beta1/networkpolicies` in kube-apiserver
- v1.7 version of Network Policy is GA, API version is `networking.k8s.io/v1`
- Added support for **Egress** and **IPBlock** in v1.8
- Network plugins support Network Policy, such as Calico, Romana, Weave Net, and trireme, refer to [here] (../plugins/network-policy.md)

## API version comparison table

| Kubernetes Version | Networking API Version |
| --------------- | -------------------- |
| v1.5-v1.6       | extensions/v1beta1   |
| v1.7 + | networking.k8s.io/v1 |

## Network Strategy

### Namespace isolation

By default, all Pods are all passable. Each Namespace can be configured with a separate network policy to isolate traffic between Pods.

The v1.7+ version is used as the default network policy by creating a Network Policy that matches all Pods, such as rejecting Ingress communication between all Pods by default.

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

The default policy for rejecting Egress communication between all Pods is

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

Even the default policy of rejecting Ingress and Egress communication between all Pods is

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

The default policy for allowing Ingress communication between all Pods is

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
```

By default, the policy for allowing Egress communication between all Pods is

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
```

The v1.6 version uses Annotation to isolate traffic between all Pods in the namespace, including external traffic to all Pods in the namespace and traffic between the internal Pods of the namespace:

```sh
kubectl annotate ns <namespace> "net.beta.kubernetes.io/network-policy={\"ingress\": {\"isolation\": \"DefaultDeny\"}}"
```

### Pod isolation

Control traffic between Pods by using tag selectors, including namespaceSelector and podSelector. For example, the following Network Policy

- Allow Pods with the `role=frontend` tag in the default namespace to access port 6379 with the `role=db` tag Pod in the default namespace
- Allow all Pods in the namespace with the `project=myprojects` tag to access port 6379 with the `role=db` tag Pod in the default namespace

`` Yaml
# v1.6 and older versions should use extensions/v1beta1
# apiVersion: extensions/v1beta1
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: tcp
      port: 6379
```

Another strategy for enabling both Ingress and Egress communication is

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

It is used to isolate Pods with the `role=db` tag in the default namespace:

- Allow Pods with the `role=frontend` tag in the default namespace to access port 6379 with the `role=db` tag Pod in the default namespace
- Allow all Pods in the namespace with the `project=myprojects` tag to access port 6379 with the `role=db` tag Pod in the default namespace
- Allow Pods with the `role=db` tag in the default namespace to access the TCP 5987 port on the `10.0.0.0/24` segment

## Simple example

Take calico as an example to see the specific usage of Network Policy.

First configure kubelet to use the CNI network plugin

```sh
cube --network-plugin = cni --cni-conf-dir = / etc / cni / net.d --cni-bin-dir = / opt / cni / bin ...
```

Install the calio network plugin

```sh
# Note that modifying CIDR needs to be consistent with k8s pod-network-cidr. The default is 192.168.0.0/16.
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

First deploy an nginx service

```sh
$ kubectl run nginx --image=nginx --replicas=2
deployment "nginx" created
$ kubectl expose deployment nginx --port=80
service "nginx" exposed
```

At this point, other Pods can access the nginx service.

```sh
$ kubectl get svc,pod
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
svc/kubernetes              10.100.0.1    <none>        443/TCP    46m
svc/nginx                   10.100.0.16   <none>        80/TCP     33s

NAME                        READY         STATUS        RESTARTS   AGE
po/nginx-701339712-e0qfq    1/1           Running       0          35s
po/nginx-701339712-o00ef    1/1           Running       0

$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
/ #
```

After turning on the DefaultDeny Network Policy of the default namespace, other Pods (including outside the namespace) cannot access nginx:

```sh
$ cat default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress

$ kubectl create -f default-deny.yaml

$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
wget: download timed out
/ #
```

Finally create a network policy that runs Pod access with `access=true`

```sh
$ cat nginx-policy.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"

$ kubectl create -f nginx-policy.yaml
networkpolicy "access-nginx" created

# Pod without access=true tag still cannot access nginx service
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
wget: download timed out
/ #


# and Pod with access=true tag can access nginx service
$ kubectl run busybox --rm -ti --labels="access=true" --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
/ #
```

Finally open the external access to the nginx service:

```sh
$ cat nginx-external-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: front-end-access
  namespace: sock-shop
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - ports:
        - protocol: TCP
          port: 80

$ kubectl create -f nginx-external-policy.yaml
```

## scenes to be used

### Prohibit access to specified services

```sh
kubectl run web --image=nginx --labels app=web,env=prod --expose --port 80
```

![](images/15022447799137.jpg)

Network strategy

`` Yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
      env: prod
```

### Only allow Pod access service

```sh
kubectl run apiserver --image=nginx --labels app=bookstore,role=api --expose --port 80
```

![](images/15022448622429.jpg)

Network strategy

`` Yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: bookstore
      role: api
  ingress:
  - from:
      - podSelector:
          matchLabels:
            app: bookstore
```

### Disable mutual access between all Pods in the namespace

![](images/15022451724392.gif)

`` Yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
```

### Prohibit other namespace access services

```sh
kubectl create namespace secondary
kubectl run web --namespace secondary --image=nginx \
    --labels=app=web --expose --port 80
```

![](images/15022452203435.gif)

Network strategy

`` Yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: secondary
  name: web-deny-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

### Only allow namespace access to the service

```sh
kubectl run web --image=nginx \
    --labels=app=web --expose --port 80
```

![](images/15022453441751.gif)

Network strategy

`` Yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-prod
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: production
```

### Allow external network access service

```sh
kubectl run web --image=nginx --labels=app=web --port 80
kubectl expose deployment/web --type=LoadBalancer
```

![](images/15022454444461.gif)

Network strategy

`` Yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-external
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - ports:
    - port: 80
    from: []
```

## Reference Document

- [Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [Securing Kubernetes Cluster Networking](https://ahmet.im/blog/kubernetes-network-policy/)
- [Kubernetes Network Policy Recipes](https://github.com/ahmetb/kubernetes-networkpolicy-tutorial)
