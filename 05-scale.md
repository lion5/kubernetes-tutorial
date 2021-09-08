# Module 5 - Scale Your App

The goal of this interactive scenario is to scale a deployment with kubectl scale and to see the load balancing in action.

## Scaling a deployment

To list your deployments use the get deployments command:

`kubectl get deployments`

The output should be similar to:

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           11m
```

We should have 1 Pod. This shows:
- `NAME` lists the names of the Deployments in the cluster.
- `READY` shows the ratio of CURRENT/DESIRED replicas
- `UP-TO-DATE` displays the number of replicas that have been updated to achieve the desired state.
- `AVAILABLE` displays how many replicas of the application are available to your users.
- `AGE` displays the amount of time that the application has been running.

To see the ReplicaSet created by the Deployment, run:

`kubectl get rs`

Notice that the name of the ReplicaSet is always formatted as `[DEPLOYMENT-NAME]-[RANDOM-STRING]`.
The random string is randomly generated and uses the pod-template-hash as a seed.

Two important columns of this command are:

- `DESIRED` displays the desired number of replicas of the application, which you define when you create the Deployment. This is the desired state.
- `CURRENT` displays how many replicas are currently running.

Next, let’s scale the Deployment to 4 replicas.
We’ll use the `kubectl scale` command, followed by the deployment type, name and desired number of instances:

`kubectl scale deployments/kubernetes-bootcamp --replicas=4`

To list your Deployments once again, use get deployments:

`kubectl get deployments`

The change was applied, and we have 4 instances of the application available.
Next, let’s check if the number of Pods changed:

`kubectl get pods -o wide`

There are 4 Pods now, with different IP addresses.
The change was registered in the deployment events log.
To check that, use the describe command:

`kubectl describe deployments/kubernetes-bootcamp`

You can also view in the output of this command that there are 4 replicas now.

## Load Balancing

Let’s check that the Service is load-balancing the traffic.
To find out the exposed IP and Port we can use the describe service as we learned in the previous Module:

`kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080`

Create an environment variable called `NODE_PORT` that has a value as the Node port:

`export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o jsonpath='{.spec.ports[0].nodePort}');echo NODE_PORT=$NODE_PORT`

To get the public IP of the node run:

`export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}');echo NODE_IP=$NODE_IP`

Next, we’ll do a `curl` to the exposed IP and port.
Execute the command multiple times:

`curl $NODE_IP:$NODE_PORT`

We hit a different Pod with every request.
This demonstrates that the load-balancing is working.

Hint: If you get an error that the `curl` URL has a bad/illegal format or missing URL, please make sure that both `$NODE_IP` and `$NODE_PORT` environment variables are set to a value.

## Scale Down

To scale down the Service to 2 replicas, run again the scale command:

`kubectl scale deployments/kubernetes-bootcamp --replicas=2`

List the Deployments to check if the change was applied with the get deployments command:

`kubectl get deployments`

The number of replicas decreased to 2.
List the number of Pods, with get pods:

`kubectl get pods -o wide`

This confirms that 2 Pods were terminated.

Adapted from https://kubernetes.io/docs/tutorials/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
