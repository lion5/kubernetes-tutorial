# Module 2 - Deploy an app

The goal of this scenario is to help you deploy and interact with your first app on Kubernetes using `kubectl`.

Again, to view the nodes in the cluster, run the `kubectl get nodes` command:

```shell
kubectl get nodes
```

Here, we see the available nodes.
Kubernetes will choose where to deploy our application based on node available resources.

## Deploy our app

Let’s deploy our first app on Kubernetes with the `kubectl create deployment` command.
We need to provide the deployment name and app image location (include the full repository url for images hosted outside Docker hub).

```shell
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

Great!
You just deployed your first application by creating a deployment.
This performed a few things for you:

- searched for a suitable node where an instance of the application could be run
- scheduled the application to run on that node
- configured the cluster to reschedule the instance on a new node when needed

To list your deployments use the get deployments command:

```shell
kubectl get deployments
```

We see that there is 1 deployment running a single instance of your app.
The instance is running inside a Docker container on a node.

## View our app

Pods that are running inside Kubernetes are running on a private, isolated network.
By default they are visible from other pods and services within the same Kubernetes cluster, but not outside that network.
When we use `kubectl`, we're interacting through an API endpoint to communicate with our application.

We will cover other options on how to expose your application outside the Kubernetes cluster in Module 4.

The `kubectl` command can create a proxy that will forward communications into the cluster-wide, private network.
The proxy can be terminated by pressing control-C and won't show any output while its running.

Open a __second terminal window__ to run the proxy:

```shell
kubectl proxy
```

We now have a connection between our host (the online terminal) and the Kubernetes cluster.
The proxy enables direct access to the API from these terminals.
Switch back to the first terminal window for the subsequent commands while keeping the window with the proxy open.

You can see all those APIs hosted through the proxy endpoint.
For example, we can query the Kubernetes version directly through the API using the `curl` command:

```shell
curl http://localhost:8001/version
```

Note: Check the top of the terminal.
The proxy was run in a new tab (Terminal 2), and the recent commands were executed the original tab (Terminal 1).
The proxy still runs in the second tab, and this allowed our `curl` command to work using localhost:8001.

If Port 8001 is not accessible, ensure that the kubectl proxy started above is running.

The API server will automatically create an endpoint for each pod, based on the pod name, that is also accessible through the proxy.

First we need to get the pod name, and we'll store it in the environment variable `POD_NAME`:

```shell
export POD_NAME=$(kubectl get pods -o jsonpath='{.items[0].metadata.name}');echo Name of the Pod: $POD_NAME
```

We also need the current namespace of your group:

```shell
export GROUP_NS=$(kubectl config view --minify -o=jsonpath='{..namespace}');echo Group namespace: $GROUP_NS
```

You can access the Pod through the API by running:

```shell
curl http://localhost:8001/api/v1/namespaces/$GROUP_NS/pods/$POD_NAME/
```

Adapted from https://kubernetes.io/docs/tutorials/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
