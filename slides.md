---
title: From 0 to Continuous Deployment with Kubernetes, Helm and GitLab
theme: moon
highlight-theme: github
separator: ------
verticalSeparator: ---
---

## From 0 to
## Continuous Deployment

**with Kubernetes, Helm and GitLab**

<small>by Stefano Torresi</small>

------

### Continuous Integration, Delivery and Deployment

- Merge developer changes into a shared mainline
- Automate integration checks
- Keep mainline in a production-ready state
- Deploy mainline automatically

---

### But why?

- Enforce best practices
- Grant confidence and safety
- Foster collaboration
- Collect code metrics

_"Release early, release often"_

Notes:
See "The Cathedral and the Bazaar" by Eric S. Raymond

---

## The pipeline

![](assets/pipeline.png)

---

## Let's build something

_Demo_

Notes:
```shell
mkdir k8s-cd-demo
git init
git remote add origin git@gitlab.com:stefanotorresi/k8s-cd-demo.git
```

------

## Kubernetes

- Container orchestration platform
- Infrastructure abstraction
- Automates container coordination and management
- API driven, declarative, stateful

Notes:
- Born from Google's Borg, which was written in C++ instead of Go.
- Donated to CNCF in 2014
- The API is fully transparent, there are no hidden internal details. This allows a very high degree of flexibility.

---

### Kubernetes Architecture

![](assets/k8s-arch.png)

Notes:
- Losely-coupled, master-worker
- Cluster centric, as opposed to how Docker Swarm is designed, which scales out from a single Docker host and then incrementally joining others.
- Master is a.k.a. the cluster control plane.
- etcd is used as the central cluster data storage.
- The `cloud-controller-manager` interacts with cloud specific details via dedicated controllers, when hosting Kubernetes on a cloud provider.
- Minions are handled by the `Kubelet` agent
- `Kube-Proxy` handles the networking abstractions.
- Master components are usually all run on the same node, which is separated from worker nodes, but this is ultimately an implementation detail. Minikube, for example, runs everything on one single node.
- HA deployments of the Master involve clustering etcd and load balancing the API server.

---

![](assets/i-can-has-clasterz.jpg)

---

## Caveat Emptor

- Very high cognitive overhead
- Operating a control plane is hard
- Easy to learn, hard to master
- Multi-tenancy is a WIP

Notes:
Hard multi-tenancy can be achieved by having one cluster per tenant

---

## Managed master to the rescue

AKS, GKE, EKS, DOKS, ICKS... all the Ks!!

![](assets/minions-rejoice.gif)

---

## Managed master to the rescue

```shell
gcloud container clusters create foobar
```

```shell
aws eks create-cluster --name foobar
```

```shell
doctl kubernetes cluster create foobar
```

Notes:
Skipping CLI options for brevity, but AWS is actually much more cumbersome than that.

---

_Demo time_

Notes:
```shell
doctl kubernetes cluster create demo \
    --region fra1 --count 2 --wait

kubectx do-fra1-demo

watch kubectl get nodes

kubectl create deployment hello-world \
    --image=

kubectl expose deployment hello-world \
    --port=80  \
    --target-port=8080 \
    --type=LoadBalancer

kubectl get deployment hello-world -o yaml
kubectl get service hello-world -o yaml
```

---

#### Whole lotta APIs

Container, Pod, Deployments, ReplicaSet, ReplicationController, StatefulSet, DaemonSet, Job, CronJob, Endpoints, Ingress, Service, ConfigMap, Secret, StorageClass, Volume, PersistentVolumeClaim, VolumeAttachment

and many others.

Notes:

[API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13)
[OpenAPI spec](https://github.com/kubernetes/kubernetes/tree/master/api/openapi-spec)

- Pods are ephemeral.
- Containers inside a Pod are co-located and run in a shared context, e.g. they share IP address and IPC.
- Deployment, ReplicaSet, DaemonSet, StatefulSet, Job are all controllers.
- Controllers provide a declarative approach to their workflow: the user declares a desired state, the controller ensures that state is present.
- Services provide access to Pods, in spite these latter being ephemeral.
- Ingresses provide external access to services, featuring load balancing, SSL termination and name-based virtual hosting.
- The spec is big: 100.000 lines long!

---

## You can do it

- Documentation is excellent
- Plenty of resources available
- Community is huge already and vibrant

---

## The YAMLPOCALYPSE problem

**Deployment example:**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: null
  generation: 1
  labels:
    app: hello-world
  name: hello-world
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/hello-world
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello-world
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-world
    spec:
      containers:
      - image: registry.gitlab.com/stefanotorresi/k8s-cd-demo:56487b78
        imagePullPolicy: IfNotPresent
        name: k8s-cd-demo
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
```

---

## The YAMLPOCALYPSE problem

**Service example:**

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: hello-world
  name: hello-world
  selfLink: /api/v1/namespaces/default/services/hello-world
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32276
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hello-world
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
```
</div>

------

## Helm

Kubernetes on steroids:

- Package and dependency manager
- Template engine
- Release tool

---

### Helm Architecture

- _Chart_: bundled application manifest
- _Release_: running instance of a _Chart_
- ```helm``` CLI: gRPC client, manages _Charts_
- _Tiller_: gRPC server, manages _Releases_

Notes:
[interesting blog post to point people at](https://medium.com/@gajus/the-missing-ci-cd-kubernetes-component-helm-package-manager-1fe002aac680)

---

### Helm Architecture

![](assets/helm-arch.png)

---

_Demo time_

Notes:
```shell
kubectl apply -f tiller-rbac.yaml

helm init --service-account tiller

helm version

helm inspect stable/external-dns | less

helm install stable/external-dns \
  --name external-dns \
  --namespace kube-system \
  --set provider=digitalocean \
  --set extraEnv[0].name=DO_TOKEN \
  --set extraEnv[0].value=$DO_TOKEN \
  --set rbac.create=true \
  --set rbac.serviceAccountName=external-dns \
  --set policy=sync

kubectl annotate service hello-world external-dns.alpha.kubernetes.io/hostname=k8s-cd-demo.torresi.io

helm install stable/nginx-ingress \
  --name nginx-ingress  \
  --namespace kube-system \
  --set-string controller.config.disable-ipv6=true
```

------

## That's all folks

Questions and feedback are welcome!

- Twitter: [@storresi](https://twitter.com/storresi)
- Email: [stefano@torresi.io](mailto:stefano@torresi.io)

**Thanks**