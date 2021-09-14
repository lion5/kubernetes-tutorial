# Module 10 - Deploying a stateless application

This tutorial shows you how to build and deploy a simple _(not production
ready)_, multi-tier web application using Kubernetes and
[Docker](https://www.docker.com/). This example consists of the following
components:

* A single-instance [Redis](https://www.redis.io/) to store guestbook entries
* Multiple web frontend instances

__Objectives__

* Start up a Redis leader.
* Start up two Redis followers.
* Start up the guestbook frontend.
* Expose and view the Frontend Service.
* Clean up.

## Start up the Redis Database

The guestbook application uses Redis to store its data.

### Creating the Redis Deployment

The manifest file, included below, specifies a Deployment controller that runs a single replica Redis Pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      containers:
      - name: leader
        image: "docker.io/redis:6.0.5"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

1. Apply the Redis Deployment from the `redis-leader-deployment.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/redis-leader-deployment.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-deployment.yaml
   ```

1. Query the list of Pods to verify that the Redis Pod is running:

   ```shell
   kubectl get pods
   ```

   The response should be similar to this:

   ```
   NAME                           READY   STATUS    RESTARTS   AGE
   redis-leader-fb76b4755-xjr2n   1/1     Running   0          13s
   ```

1. Run the following command to view the logs from the Redis leader Pod:

   ```shell
   kubectl logs -f deployment/redis-leader
   ```

### Creating the Redis leader Service

The guestbook application needs to communicate to the Redis to write its data.
You need to apply a [Service](/docs/concepts/services-networking/service/) to
proxy the traffic to the Redis Pod. A Service defines a policy to access the
Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader
    tier: backend
```

1. Apply the Redis Service from the following `redis-leader-service.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/redis-leader-service.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-service.yaml
   ```

1. Query the list of Services to verify that the Redis Service is running:

   ```shell
   kubectl get service
   ```

   The response should be similar to this:

   ```
   NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
   redis-leader   ClusterIP   10.103.78.24 <none>        6379/TCP   16s
   ```

### Set up Redis followers

Although the Redis leader is a single Pod, you can make it highly available
and meet traffic demands by adding a few Redis followers, or replicas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: follower
        tier: backend
    spec:
      containers:
      - name: follower
        image: gcr.io/google_samples/gb-redis-follower:v2
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

1. Apply the Redis Deployment from the following `redis-follower-deployment.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/redis-follower-deployment.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-deployment.yaml
   ```

1. Verify that the two Redis follower replicas are running by querying the list of Pods:

   ```shell
   kubectl get pods
   ```

   The response should be similar to this:

   ```
   NAME                             READY   STATUS    RESTARTS   AGE
   redis-follower-dddfbdcc9-82sfr   1/1     Running   0          37s
   redis-follower-dddfbdcc9-qrt5k   1/1     Running   0          38s
   redis-leader-fb76b4755-xjr2n     1/1     Running   0          11m
   ```

### Creating the Redis follower service

The guestbook application needs to communicate with the Redis followers to
read data. To make the Redis followers discoverable, you must set up another
[Service](/docs/concepts/services-networking/service/).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: follower
    tier: backend
```

1. Apply the Redis Service from the following `redis-follower-service.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/redis-follower-service.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-service.yaml
   ```

1. Query the list of Services to verify that the Redis Service is running:

   ```shell
   kubectl get service
   ```

   The response should be similar to this:

   ```
   NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
   redis-follower   ClusterIP   10.110.162.42   <none>        6379/TCP   9s
   redis-leader     ClusterIP   10.103.78.24    <none>        6379/TCP   6m10s
   ```

## Set up and Expose the Guestbook Frontend

Now that you have the Redis storage of your guestbook up and running, start
the guestbook web servers. Like the Redis followers, the frontend is deployed
using a Kubernetes Deployment.

The guestbook app uses a PHP frontend. It is configured to communicate with
either the Redis follower or leader Services, depending on whether the request
is a read or a write. The frontend exposes a JSON interface, and serves a
jQuery-Ajax-based UX.

### Creating the Guestbook Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
        app: guestbook
        tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80
```

1. Apply the frontend Deployment from the `frontend-deployment.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/frontend-deployment.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
   ```

1. Query the list of Pods to verify that the three frontend replicas are running:

   ```shell
   kubectl get pods -l app=guestbook -l tier=frontend
   ```

   The response should be similar to this:

   ```
   NAME                        READY   STATUS    RESTARTS   AGE
   frontend-85595f5bf9-5tqhb   1/1     Running   0          47s
   frontend-85595f5bf9-qbzwm   1/1     Running   0          47s
   frontend-85595f5bf9-zchwc   1/1     Running   0          47s
   ```

### Creating the Frontend Service

The `Redis` Services you applied is only accessible within the Kubernetes
cluster because the default type for a Service is
[ClusterIP](/docs/concepts/services-networking/service/#publishing-services-service-types).
`ClusterIP` provides a single IP address for the set of Pods the Service is
pointing to. This IP address is accessible only within the cluster.

If you want guests to be able to access your guestbook, you must configure the
frontend Service to be externally visible, so a client can request the Service
from outside the Kubernetes cluster. However a Kubernetes user you can use
`kubectl port-forward` to access the service even though it uses a
`ClusterIP`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  ports:
    # the port that this service should serve on
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```

1. Apply the frontend Service from the `frontend-service.yaml` file:

   <!---
   for local testing of the content via relative file path
   kubectl apply -f ./content/en/examples/application/guestbook/frontend-service.yaml
   -->

   ```shell
   kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
   ```

1. Query the list of Services to verify that the frontend Service is running:

   ```shell
   kubectl get services
   ```

   The response should be similar to this:

   ```
   NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
   frontend         ClusterIP   10.97.28.230    <none>        80/TCP     19s
   redis-follower   ClusterIP   10.110.162.42   <none>        6379/TCP   5m48s
   redis-leader     ClusterIP   10.103.78.24    <none>        6379/TCP   11m
   ```

### Viewing the Frontend Service via `kubectl port-forward`

1. Run the following command to forward port `8080` on your local machine to port `80` on the service.

   ```shell
   kubectl port-forward svc/frontend 8080:80
   ```

   The response should be similar to this:

   ```
   Forwarding from 127.0.0.1:8080 -> 80
   Forwarding from [::1]:8080 -> 80
   ```

1. load the page [http://localhost:8080](http://localhost:8080) in your browser to view your guestbook.

Try adding some guestbook entries by typing in a message, and clicking Submit.
The message you typed appears in the frontend. This message indicates that
data is successfully added to Redis through the Services you created earlier.

Use `CTRL-C` to stop the proxy.

### Configure a public Ingress for the Service

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /group-1
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

Change your groups path name and deploy:

```shell
kubectl apply -f path_to_ingress_file
```

### Viewing the Frontend Service via `Ingress`

Retrieve the Ingress resource and wait for the external IP address/DNS to view your Guestbook:

1. Run the following command to get the IP address for the frontend Service.

   ```shell
   kubectl get ingress guestbook-ingress
   ```

   The response should be similar to this:

```
NAME                CLASS   HOSTS   ADDRESS                                                                         PORTS   AGE
guestbook-ingress   nginx   *       a30be3be54566445c86ca3a56e736230-788d26b13fc29c22.elb.us-east-1.amazonaws.com   80      5m3s
```

1. Copy the external IP/DNS address, and load the page in your browser to view your guestbook. Make sure to append your path postfix, e.g., `/group-1`.

```
a30be3be54566445c86ca3a56e736230-788d26b13fc29c22.elb.us-east-1.amazonaws.com/group-1
```

## Cleanup

Deleting the Deployments and Services also deletes any running Pods. Use
labels to delete multiple resources with one command.

1. Run the following commands to delete all Pods, Deployments, and Services.

   ```shell
   kubectl delete deployment -l app=redis
   kubectl delete service -l app=redis
   kubectl delete deployment frontend
   kubectl delete service frontend
   ```

   The response should look similar to this:

```
deployment.apps "redis-follower" deleted
deployment.apps "redis-leader" deleted
deployment.apps "frontend" deleted
service "frontend" deleted
```

1. Query the list of Pods to verify that no Pods are running:

   ```shell
   kubectl get pods
   ```

   The response should look similar to this:

   ```
   No resources found in default namespace.
   ```

Adapted from https://kubernetes.io/docs/tutorials/stateless-application/guestbook/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
