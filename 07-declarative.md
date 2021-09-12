# Module 7 - Declarative Configuration

Kubernetes objects can be created, updated, and deleted by storing multiple object configuration files in a directory and using `kubectl apply` to recursively create and update those objects as needed.
This method retains writes made to live objects without merging the changes back into the object configuration files.
`kubectl diff` also gives you a preview of what changes `apply` will make.

## How to create objects

Use `kubectl apply` to create all objects, except those that already exist,
defined by configuration files in a specified directory:

```shell
kubectl apply -f <directory>/
```

This sets the `kubectl.kubernetes.io/last-applied-configuration: '{...}'`
annotation on each object. The annotation contains the contents of the object
configuration file that was used to create the object.

Here's an example of an object configuration file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Run `kubectl diff` to print the object that will be created:

```shell
kubectl diff -f https://k8s.io/examples/application/simple_deployment.yaml
```

`diff` performs a server-side apply request in dry-run mode, to evaluate the outcome of the operation.

Create the object using `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the `kubectl.kubernetes.io/last-applied-configuration` annotation
was written to the live configuration, and it matches the configuration file.

## How to update objects

You can also use `kubectl apply` to update all objects defined in a directory, even
if those objects already exist.
This approach accomplishes the following:

1. Sets fields that appear in the configuration file in the live configuration.
2. Clears fields removed from the configuration file in the live configuration.

```shell
kubectl diff -f <directory>/
kubectl apply -f <directory>/
```

Here's an example configuration file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Create the object using `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the `kubectl.kubernetes.io/last-applied-configuration` annotation
was written to the live configuration, and it matches the configuration file.

Directly update the `replicas` field in the live configuration by using `kubectl scale`.
This does not use `kubectl apply`:

```shell
kubectl scale deployment/nginx-deployment --replicas=2
```

Print the live configuration using `kubectl get`:

```shell
kubectl get deployment nginx-deployment -o yaml
```

The output shows that the `replicas` field has been set to 2, and the `last-applied-configuration`
annotation does not contain a `replicas` field.

Update the `simple_deployment.yaml` configuration file to change the image from
`nginx:1.14.2` to `nginx:1.16.1`, and delete the `minReadySeconds` field:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # update the image
        ports:
        - containerPort: 80
```

Apply the changes made to the configuration file:

```shell
kubectl diff -f https://k8s.io/examples/application/update_deployment.yaml
kubectl apply -f https://k8s.io/examples/application/update_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/update_deployment.yaml -o yaml
```

The output shows the following changes to the live configuration:

* The `replicas` field retains the value of 2 set by `kubectl scale`.
  This is possible because it is omitted from the configuration file.
* The `image` field has been updated to `nginx:1.16.1` from `nginx:1.14.2`.
* The `last-applied-configuration` annotation has been updated with the new image.
* The `minReadySeconds` field has been cleared.
* The `last-applied-configuration` annotation no longer contains the `minReadySeconds` field.

## How to delete objects

There are two approaches to delete objects managed by `kubectl apply`.
`kubectl apply` will not delete resources that are removed from the YAML file by itself.

### Recommended: `kubectl delete -f <filename>`

Manually deleting objects using the imperative command is the recommended
approach, as it is more explicit about what is being deleted, and less likely
to result in the user deleting something unintentionally.

```shell
kubectl delete -f https://k8s.io/examples/application/update_deployment.yaml
```

As an alternative to `kubectl delete`, you can use `kubectl apply` to identify objects to be deleted after their configuration files have been removed from the directory. Apply with `--prune` queries the API server for all objects matching a set of labels, and attempts to match the returned live object configurations against the object configuration files. If an object matches the query, and it does not have a configuration file in the directory, and it has a last-applied-configuration annotation, it is deleted.

Only use this if you know what you are doing!

Adapted from https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/ licenced under [Creative Commons Attribution 4.0 International](https://github.com/kubernetes/website/blob/main/LICENSE).
