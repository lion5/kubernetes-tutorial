# Module 4 - Expose your app publicly

In this scenario you will learn how to expose Kubernetes applications outside the cluster using the `kubectl expose` command.
You will also learn how to view and apply labels to objects with the `kubectl label` command.

## Create a new service

Let’s verify that our application is running.
We’ll use the `kubectl get` command and look for existing Pods:

```shell
kubectl get pods
```

Next, let’s list the current services from our cluster:

```shell
kubectl get services
```

Currently, there should be no services running in your namespace.

You can list all services in all namespaces in the K8s cluster via:

```shell
kubectl get services -A
```

For example, we have a service called kubernetes for the K8s API server that is created by default when the cluster is started.

To create a new service and expose it to external traffic we’ll use the expose command with NodePort as parameter.

```shell
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

Let’s run again the get services command:

```shell
kubectl get services
```

We have now a running Service called kubernetes-bootcamp.
Here, we see that the Service is of type NodePort and received a unique cluster-IP, an internal port and a NodePort forwarding.

To find out what port was opened externally (by the NodePort option) we’ll run the describe service command:

```shell
kubectl describe services/kubernetes-bootcamp
```

Create an environment variable called `NODE_PORT` that has the value of the Node port assigned:

```shell
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o jsonpath='{.spec.ports[0].nodePort}');echo NODE_PORT=$NODE_PORT
```

To get the public IP of the node run:

```shell
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}');echo NODE_IP=$NODE_IP
```

Now we can test that the app is exposed outside of the cluster using `curl`, the IP of the Node and the externally exposed port:

```shell
curl $NODE_IP:$NODE_PORT
```

And we get a response from the server.
The Service is exposed.

Hint: If you get an error that the `curl` URL has a bad/illegal format or missing URL, please make sure that both `$NODE_IP` and `$NODE_PORT` environment variables are set to a value.

## Using labels

The Deployment created automatically a label for our Pod.
With describe deployment command you can see the name of the label:

```shell
kubectl describe deployment
```

Let’s use this label to query our list of Pods.
We’ll use the kubectl get pods command with `-l` as a parameter, followed by the label values:

```shell
kubectl get pods -l app=kubernetes-bootcamp
```

You can do the same to list the existing services:

```shell
kubectl get services -l app=kubernetes-bootcamp
```

Get the name of the Pod and store it in the `POD_NAME` environment variable:

```shell
export POD_NAME=$(kubectl get pods -o jsonpath='{.items[0].metadata.name}');echo Name of the Pod: $POD_NAME
```

To apply a new label, we use the label command followed by the object type, object name and the new label:

```shell
kubectl label pods $POD_NAME version=v1
```

This will apply a new label to our Pod (we pinned the application version to the Pod), and we can check it with the `kubectl describe pod` command:

```shell
kubectl describe pods $POD_NAME
```

We see here that the label is attached now to our Pod.
And we can query now the list of pods using the new label:

```shell
kubectl get pods -l version=v1
```

And we see the Pod.

## Deleting a service

To delete Services you can use the `kubectl delete service` command.
Labels can be used also here:

```shell
kubectl delete service -l app=kubernetes-bootcamp
```

Confirm that the service is gone:

```shell
kubectl get services
```

This confirms that our Service was removed.
To confirm that route is not exposed anymore you can `curl` the previously exposed IP and port:

```shell
curl $NODE_IP:$NODE_PORT
```

This proves that the app is not reachable anymore from outside of the cluster.

Hint: If you get an error that the `curl` URL has a bad/illegal format or missing URL, please make sure that both `$NODE_IP` and `$NODE_PORT` environment variables are set to a value.

You can confirm that the app is still running with a `curl` inside the pod:

```shell
kubectl exec -ti $POD_NAME -- curl localhost:8080
```

Here, we see that the application is up.
This is because the deployment is managing the application.
To shut down the application, you would need to delete the deployment as well.

Adapted from https://kubernetes.io/docs/tutorials/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
