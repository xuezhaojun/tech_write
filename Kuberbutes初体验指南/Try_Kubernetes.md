# Try Kubernetes

Created: Apr 9, 2021 3:41 PM

### Create a cluster in container using kind

Create a cluster named "hello" using config file `hellokind.yaml`

```bash
kind create cluster --name="hello" --config=hellokind.yaml
```

Use `kind get` to check if cluster is created.

```bash
kind get clusters
hello
```

Kind create cluster **in a container**, so we can see a new container named "hello-control-plane" in docker:

```bash
docker ps

CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                                                NAMES
115d351f2199   kindest/node:v1.20.2   "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes   127.0.0.1:53082->6443/tcp, 0.0.0.0:8080->30950/tcp   hello-control-plane
```

Then load our go-web-server Image into nodes: (you need to build an image first)

```bash
kind load docker-image hellokube:1.0.0 --name hello
```

![Untitled.png](https://ae01.alicdn.com/kf/Uba344f2de2054e3e822ab958239fd36bl.png)

For more information about kind: [https://kind.sigs.k8s.io/docs](https://kind.sigs.k8s.io/docs)

---

### Run containers

To run a container, at first we need to know **pod, node and cluster**:

![Untitled 1.png](https://ae04.alicdn.com/kf/Ue2fa301315f34b16af5b15f45898d8d7B.png)

Some basic concepts and rules:

- Containers(one or more) run in a pod
- Pods are the **smallest** deployable units of computing that you can create and manage in Kubernetes.
    - Which means we can't create and manage containers directly
- A Pod runs in a Node

Basically, Pods **carry our workload** and **nodes** bind with a machine(virtual or physical). Thus we can move pods from a "poor" node to a "rich" node(with more compute power) if we want.

For more informations about Pod and Node:

- Pod [https://kubernetes.io/docs/concepts/workloads/pods/](https://kubernetes.io/docs/concepts/workloads/pods/)
- Node [https://kubernetes.io/docs/concepts/architecture/nodes/](https://kubernetes.io/docs/concepts/architecture/nodes/)

Now we are going to run containers(in pods) in a cluster, **we want to run two functionally equal pods on two nodes** in case of one node(as we said node is a machine) was broken by accident. 

We want to **check what current status** is and **make it to the status we expected**, in the previous case, we expect **two pods on two nodes.** 

In Kubernetes we use **workload resources and controller** to do this.  There are different kind of resources to match different exceptions, a basic one is **Deployment.** We use a YAML file to describe a Deployment: 

(same code in `hellodep.yaml` )

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellodep
  labels:
    app: hellodep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellokube
  template:
    metadata:
      labels:
        app: hellokube
    spec:
      containers:
      - name: hellokube
        image: hellokube:1.0.0
        ports:
        - containerPort: 8080
```

 ****`spec` represent for specification(what we expect), so this YAML file means we want `2 replicate` pods. And `template`  part means we use image `hellokube:1.0.0` to run container in the pods. Run command:

```bash
kubectl apply -f hellodep.yam
```

There are two new pods in our cluster:

```bash
kubectl get pods

NAME                        READY   STATUS    RESTARTS   AGE
hellodep-84548c9c8f-hjd4g   1/1     Running   0          36s
hellodep-84548c9c8f-vvrp9   1/1     Running   0          36s
```

For more informations about **workload resource**: [https://kubernetes.io/docs/concepts/workloads/](https://kubernetes.io/docs/concepts/workloads/)

---

### What happened behind

![Untitled 2.png](https://ae02.alicdn.com/kf/U2d80e5a1fcfa4a67801c80bf4a229cdfm.png)

![Untitled 3.png](https://ae05.alicdn.com/kf/Ubfa331384cf84984b4bcc7112ab43946o.png)

It's not what exactly happened, but basically we should know:

- **etcd** store deployment
- **kubelet** on node create/delete pods
- **controller** check current status and tell kubernetes what to do next
- all components communicate through **apiserver** (via broatcast and watch events)

（we simplified the case by ignoring the **scheduler** part）

For more information about :

- architecture: [https://kubernetes.io/zh/docs/concepts/architecture/](https://kubernetes.io/zh/docs/concepts/architecture/)
- controller and events: [https://book-v1.book.kubebuilder.io/basics/what_is_a_controller.html](https://book-v1.book.kubebuilder.io/basics/what_is_a_controller.html)

---

### Expose using Service

In our hellokube server, we listen to port `8080`, so we can do a port mapping on it:

```bash
kubectl port-forward pod/hellodep-84548c9c8f-hjd4g 8080:8080
```

Now if we send a get request to port `8080`, we get a hello response:

```bash
curl http://localhost:8080
Hello Kube! 2021-04-09 17:47:04.5703924 +0000 UTC m=+5598.039692701
```

But as we said before, pods can be "moved" from one node to another frequently, and every new pod has a new ip. We want a consistent way to connect our backend server. 

We use Service can provide a IP:Port for a group of pods. When some pods are deleted or new pods joined, the IP:Port of Service **won't change**.

Service is also a kind of resources, we can use YAML to describte a Service:

(see this in helloservice.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helleservice
spec:
  type: NodePort
  ports:
  - name: http
    nodePort: 30950
    port: 8080
  selector:
    app: hellokube
```

Then we create a Service and Endpoints for this Service(Endpoints has ip for the group of pods):

```bash
kubectl apply -f helloservice.yaml

kubectl get services
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
helleservice   NodePort    10.96.89.17   <none>        8080:30950/TCP   5s

kubectl get endpoints
NAME           ENDPOINTS                         AGE
helleservice   10.244.0.5:8080,10.244.0.6:8080   23m

// test
curl http://localhost:8080
Hello Kube! 2021-04-09 17:57:53.0373083 +0000 UTC m=+6247.265377501
```

Service and Endpoints are implemented by iptables, and Iptable are maintained by Node Component - kube-proxy. Here we don't go to detail.

![Untitled 4.png](https://ae04.alicdn.com/kf/Uc772ffc8e3aa4b6bb15431b02576b246i.png)

For more information about Service : [https://kubernetes.io/zh/docs/concepts/services-networking/service/](https://kubernetes.io/zh/docs/concepts/services-networking/service/)