# Module 9 - Configure two Pods to use a shared PersistentVolume for Storage

This scenario shows you how to configure two Pods to use a shared persistent volume for storage and communicate with each other.

Here is a summary of the process:

1. You, are now taking the role of a developer / cluster user, create a
PersistentVolumeClaim that is automatically bound to a suitable
PersistentVolume.

1. You, use the default storage class to let Kubernetes provision a suitable persistent volume for you.

1. You create two Pods that use the above PersistentVolumeClaim for storage.

## Create a PersistentVolumeClaim

The next step is to create a PersistentVolumeClaim.
Pods use PersistentVolumeClaims to request physical storage.
In this exercise, you create a PersistentVolumeClaim that requests a volume of at least 1 gibibytes that can provide read-write
access for at least one Node.

Here is the configuration file for the PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Create a file with the contents and apply the PersistentVolumeClaim:

```shell
kubectl apply -f <path-to-pv-claim-file>
```

After you create the PersistentVolumeClaim, the Kubernetes control plane looks for a PersistentVolume that satisfies the claim's requirements.
If the control plane finds a suitable PersistentVolume with the same StorageClass, it binds the claim to the volume.
In our case, we did not request a storage class, so K8s will try to use the default storage class to dynamically create a persistent volume for us.

Use the following command to view the available storage classes:

```shell
kubectl get storageclass
```

Take a look at the storage class that is marked as default.

Now, let's check if a a persistent volume was created for us:

```shell
kubectl get pv
```

None, so far as it seems.
What happened?

Let's taker a look at the PersistentVolumeClaim:

```shell
kubectl get pvc task-pv-claim
```

The output should show a event `WaitForFirstConsumer` with a reason `waiting for first consumer to be created before binding`.

Ok, so the persistent volume is only created when a consumer, e.g., a Pod tries to mount the volume.

##  Creating a Pod that runs two Containers with a shared volume

The next step is to create a Pod that uses your PersistentVolumeClaim as a volume.

Create a Pod that runs two Containers.
The two containers share a Volume that they can use to communicate.
Here is the configuration file for the Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never

  volumes:
  - name: shared-data
    persistentVolumeClaim:
        claimName: task-pv-claim

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

In the configuration file, you can see that the Pod has a Volume named
`shared-data`.

Notice that the Pod's configuration file specifies a PersistentVolumeClaim, but
it does not specify a PersistentVolume. From the Pod's point of view, the claim is a volume.

The first container listed in the configuration file runs an nginx server. The mount path for the shared Volume is `/usr/share/nginx/html`.
The second container is based on the debian image, and has a mount path of
`/pod-data`. The second container runs the following command and then terminates.

    echo Hello from the debian container > /pod-data/index.html

Notice that the second container writes the `index.html` file in the root
directory of the nginx server.

Create a file with the contents and create the Pod and the two containers:

```shell
kubectl apply -f <path-to-pod-file>
```

## Check the dynamically created PersistentVolume

Let's reevaluate our pending persistent volume claim:

```shell
kubectl get pvc task-pv-claim
```

The output now shows that the PersistentVolumeClaim is bound to a new PersistentVolume (`STATUS` of `Bound`).

```
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound    pvc-863eab33-f2fb-40f0-94a7-6ff217c3d0dd   1Gi        RWO            gp2            22m
```

Now show the persistent volume:

```shell
kubectl get pv replace_with_name_of_volume
```

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pvc-863eab33-f2fb-40f0-94a7-6ff217c3d0dd   1Gi        RWO            Delete           Bound    group-6/task-pv-claim   gp2                     9m11s
```

And view the details of the dynamically provisioned resource:

```
kubectl describe pv replace_with_name_of_volume
```
    
## Observe what the containers did with the shared volume 

View information about the Pod and the Containers:

    kubectl get pod two-containers --output=yaml

Here is a portion of the output:

    apiVersion: v1
    kind: Pod
    metadata:
      ...
      name: two-containers
      namespace: default
      ...
    spec:
      ...
      containerStatuses:

      - containerID: docker://c1d8abd1 ...
        image: debian
        ...
        lastState:
          terminated:
            ...
        name: debian-container
        ...

      - containerID: docker://96c1ff2c5bb ...
        image: nginx
        ...
        name: nginx-container
        ...
        state:
          running:
        ...

You can see that the debian Container has terminated, and the nginx Container
is still running.

Get a shell to nginx Container:

    kubectl exec -it two-containers -c nginx-container -- /bin/bash

In your shell, verify that nginx is running:

    root@two-containers:/# apt-get update
    root@two-containers:/# apt-get -y install curl procps
    root@two-containers:/# ps aux

The output is similar to this:

    USER       PID  ...  STAT START   TIME COMMAND
    root         1  ...  Ss   21:12   0:00 nginx: master process nginx -g daemon off;

Recall that the debian Container created the `index.html` file in the nginx root
directory. Use `curl` to send a GET request to the nginx server:

```
root@two-containers:/# curl localhost
```

The output shows that nginx serves a web page written by the debian container:

```
Hello from the debian container
```

Leave the subshell via `exit`.

## Discussion

The primary reason that Pods can have multiple containers is to support
helper applications that assist a primary application.
Typical examples of helper applications are data pullers, data pushers, and proxies.
Helper and primary applications often need to communicate with each other.
Typically this is done through a shared filesystem, as shown in this exercise,
or through the loopback network interface, localhost. An example of this pattern is a
web server along with a helper program that polls a Git repository for new updates.

The Volume in this exercise provides a way for Containers to communicate during
the life of the Pod.
    
## Checking the persistent state of a volume

Now, let's simulate that a pod gets moved to a new node or destroyed.

Delete the Pod using kubectl delete command:

```shell
kubectl delete -f replace_with_pod_yaml_file
```

Check the status of the Pod:

```shell
kubectl get pods
```

Check that the persistent volume still exists:

```shell
kubectl get pv
```

Create a new pod with only the volume mount:

Here is the configuration file for the Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: one-container
spec:
  restartPolicy: Never

  volumes:
  - name: shared-data
    persistentVolumeClaim:
        claimName: task-pv-claim

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
```

```shell
kubectl apply -f replace_with_new_pod_yaml_file
```

Get a shell to nginx Container:

    kubectl exec -it one-container -c nginx-container -- /bin/bash

Recall that the debian Container created the `index.html` file in the volume directory. Use `curl` to send a GET request to the nginx server:

```
root@two-containers:/# curl localhost
```

The output shows that the volume was reattached and the previously written message is still persistent:

```
Hello from the debian container
```

Leave the subshell via `exit`.

## Clean up

Delete the Pod, the PersistentVolumeClaim and the PersistentVolume:

```shell
kubectl delete -f replace_with_new_pod_yaml_file
kubectl delete pvc task-pv-claim
```

The persistent volume should be automatically cleaned up by the cloud provisioner after the persistent volume claim was removed:

```shell
kubectl get pv
```

Adapted from https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
