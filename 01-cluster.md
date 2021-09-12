# Module 1 - Explore your cluster

In this scenario, we want to configure the connection of the CLI `kubectl` with our Kubernetes cluster and explore first cluster details.

## kubectl configuration

To interact with Kubernetes during this bootcamp we’ll use the command line interface, `kubectl`.

To check if `kubectl` is installed you can run the `kubectl version` command:

```shell
kubectl version
```

OK, `kubectl` is configured and we can see both, the version of the client as well as the server.
The client version is the `kubectl` version; the server version is the Kubernetes version installed on the master.
You can also see details about the build.

Type `kubectl` in the terminal to see its usage.
The common format of a kubectl command is: kubectl action resource.
This performs the specified action (like get, describe, create) on the specified resource (like node, container).
You can use `--help` after the command to get additional info about possible parameters (kubectl config --help).

Before we can access our cluster, we have to make sure that `kubectl` is correctly configured for the cluster.
To allow access to multiple clusters, `kubectl` allows to configure connections via contexts.
For every cluster connection there exists a context inside a kubeconfig configuration file.
By default, `kubectl` looks for a file named `config` in the `$HOME/.kube` directory.

To view all available contexts run:

```shell
kubectl config get-contexts
```

The currently selected context can be printed via:

```shell
kubectl config current-context
```

Make sure your current context targets `arn:aws:eks:us-east-1:935581097791:cluster/training-cluster`.
If not, switch to the correct context via: `kubectl config use-context replace_with_context_name`

To view the entire kubeconfig, enter this command:

```shell
kubectl config view
```

## Cluster details

Let’s view the cluster details.
We’ll do that by running `kubectl cluster-info`:

```shell
kubectl cluster-info
```

The command prints the address of the K8s control plane and cluster services.

To view the nodes in the cluster, run the `kubectl get nodes` command:

```shell
kubectl get nodes
```

This command shows all nodes that can be used to host our applications.
We can see all available nodes and that their status is ready (they are ready to accept applications for deployment).

We can use the output option `-o wide`, to display more detailed information for most K8s resources:

```shell
kubectl get nodes -o wide
```

Here, we can additionally see the OS image of the nodes and node IPs.

For a fully detailed view of a resource, including usage and events, we can use the `describe` command instead:

```shell
kubectl describe nodes replace_with_node_name
```

Adapted from https://kubernetes.io/docs/tutorials/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
