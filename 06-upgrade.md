# Module 6 - Update Your App

The goal of this scenario is to update a deployed application with `kubectl set image` and to rollback with the `kubectl rollout undo` command.

## Update the version of the app

To list your deployments, run the get deployments command:

```shell
kubectl get deployments
```

To list the running Pods, run:

```shell
kubectl get pods
```

To view the current image version of the app, run the `describe pods` command and look for the Image field:

```shell
kubectl describe pods
```

To update the image of the application to version 2, use the `set image` command, followed by the deployment name and the new image version:

```shell
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

The command notified the Deployment to use a different image for your app and initiated a rolling update.
Check the status of the new Pods, and view the old one terminating with the get pods command:

```shell
kubectl get pods
```

## Verify an update

First, check that the app is running.
To find the exposed IP and Port, run the describe service command:

```shell
kubectl describe services/kubernetes-bootcamp
```

Create an environment variable called NODE_PORT that has the value of the Node port assigned:

```shell
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o jsonpath='{.spec.ports[0].nodePort}');echo NODE_PORT=$NODE_PORT
```

To get the public IP of the node run:

```shell
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}');echo NODE_IP=$NODE_IP
```

Next, do a `curl` to the the exposed IP and port:

```shell
curl $NODE_IP:$NODE_PORT
```

Hint: If you get an error that the `curl` URL has a bad/illegal format or missing URL, please make sure that both `$NODE_IP` and `$NODE_PORT` environment variables are set to a value.

Every time you run the curl command, you will hit a different Pod.
Notice that all Pods are running the latest version (v2).

You can also confirm the update by running the rollout status command:

```shell
kubectl rollout status deployments/kubernetes-bootcamp
```

To view the current image version of the app, run the `describe pods` command:

```shell
kubectl describe pods
```

In the Image field of the output, verify that you are running the latest image version (v2).

## Rollback an update

Let’s perform another update, and deploy an image tagged with v10:

```shell
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
```

Use get deployments to see the status of the deployment:

```shell
kubectl get deployments
```

Notice that the output doesn't list the desired number of available Pods.
Run the get pods command to list all Pods:

```shell
kubectl get pods
```

Notice that some of the Pods have a status of ImagePullBackOff.

To get more insight into the problem, run the describe pods command:

```shell
kubectl describe pods
```

In the Events section of the output for the affected Pods, notice that the v10 image version did not exist in the repository.

To roll back the deployment to your last working version, use the `rollout undo` command:

```shell
kubectl rollout undo deployments/kubernetes-bootcamp
```

The `rollout undo` command reverts the deployment to the previous known state (v2 of the image).
Updates are versioned and you can revert to any previously known state of a deployment.

Use the get pods commands to list the Pods again:

```shell
kubectl get pods
```

Two Pods are running.
To check the image deployed on these Pods, use the `describe pods` command:

```shell
kubectl describe pods
```

The deployment is once again using a stable version of the app (v2). The rollback was successful.

Adapted from https://kubernetes.io/docs/tutorials/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
