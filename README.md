# A brief introduction to Kubernetes (k8s)

## Who am I?

* I'm Austin Vance. I have done a few things mostly coding or
managing operations teams.

* I have a history at Pivotal, EMC, Dell,
and Paypal.

* Now I have Focused Labs and we are a growing amazing team!
  * We have been working with Twenty on DevOps for Healthy Together for the last few months
  * A big part of that work is migrating workloads to Kubernetes

## 1. What is Kubernetes
> Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

- Invented by Google
- About 6 years old as OSS and 15 years at google
- Fastest growing open source project of all time

### So what?

Kubernetes runs and manages containers

There are managed Kubernetes services
- EKS
- AKS
- GKE
- Digital Ocean
- ... a lot more

Or you can deploy to your own infrastructure/locally

### Why do we like k8s?

1. Containers have become the defacto way of developing & deploying applications
1. Managing containers can be a real pain. Everything from networking to persistence is difficult when you are HA or in 
  multi node environemnt
1. It's a RESTFUL api designed for extensibility (more on that later)
1. Therotically no vendor lockin
1. Open source
1. Last but not least. It can run anything you can put in a container.

### General Architecture

#### Master Nodes
Keeping with the nautical theme, these are the port masters. They host/connect to etcd for state, run the kubernetes API, and a few other components to manage the cluster.

Depending on your installation you may not have access to the master nodes (EKS)


#### Worker Nodes
These are the ships - they hold containers, manage their own workloads and process, and let the master node know their state for scheduling. The master nodes place containers onto the worker nodes but the node manages that container.

### APIs for Developers

Kubernetes runs containers and applications by having developers develop and apply configuration. This is a core concept of kubernetes.

Applications, routing, networking, batch, Authn/z are all controlled via RESTful resources.

What matters most to developers
- [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)
  - Runs one or more containers
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
  - Manages Pods
  - Replicas to manage
  - Release strategy
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
  - Networking between Pods
  - Networking from between nodes
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
  - External networking (alb, nginx, elb, nlb...)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
  - One time finsh or retry executions
  - Can be run as a [cron](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) too

## Let's use it!

### Installing the tooling

You need the [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) cli. So first install that.

To get access to the cluster reach out to your devops team

### Running an Application

#### Deploy a POD

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
  metadata:
    labels:
      app: "2048"
  name: 2048-pod
spec:
  containers:
  - image: alexwhen/docker-2048
    imagePullPolicy: Always
    name: "2048"
    ports:
    - containerPort: 80
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF
```

Now we can see our pod running in the cluster

```bash
kubectl get pods
```

#### Deploy a Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "2048-deployment"
  namespace: "2048-game"
spec:
  selector:
    matchLabels:
      app: "2048"
  replicas: 5 # Notice replicas
  template: # Notice this is the same as the pod spec above
    metadata:
      labels:
        app: "2048"
    spec:
      containers:
      - image: alexwhen/docker-2048
        imagePullPolicy: Always
        name: "2048"
        ports:
        - containerPort: 80
EOF
```

Now let's look at our pods

```
kubectl get pods

# See the 5 pods running with a unique suffix
```

#### Deploy a service

How do we route requests to those pods? They are running but how can we get to the game?

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: "service-2048"
  namespace: "2048-game"
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort # <-
  selector:
    app: "2048" # Notice this matches the labels on the pod/deployment above
EOF
```

See our service running

```bash
kubectl get services
```

Can we route to our service, not yet but we can port forward


```bash
kubectl port-forward svc/service-2048 1337:80
```

##### How about making it public
Edit the above to use a service `type` of `LoadBalancer` this works with the AWS CNI to deploy an ELB

```bash
kubectl edit service service-2048
# Update the deployment type
kubectl get service -o wide
```

In the output we will see a dns entry for the ELB (because of our installation this ELB is on a private subnet and cannot be accessed from the internet)

So let's edit it back to a `NodePort` and deploy an ingress

#### Deploy an Ingress

Because we have some tooling in place our ingress will automatically provision an ALB with SSL from ACM and DNS entries in route 53

```bash
cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "2048-ingress"
  namespace: "2048-game"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing # should we be internal or external
  labels:
    app: 2048-ingress
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "service-2048" # map to the service
              servicePort: 80
EOF
```


## Where to next

- [Gitops](https://www.weave.works/technologies/gitops/)
- [Helm](https://helm.sh/)
- Advanced API usage


> If you're really interested I recomend the [CKAD](https://www.cncf.io/certification/ckad/) there's a course on Udemy that's normally abotu $12 https://www.udemy.com/course/certified-kubernetes-application-developer/


```text
Reference

https://github.com/kubernetes-sigs/aws-alb-ingress-controller/tree/master/docs/examples/2048
```
