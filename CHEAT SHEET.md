> Full cheat sheet can be found at official Kubernetes documentation: 
[kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

# Some useful commands and when to use them (must run on Control plane node)

## Get resources

use `kubectl get <resource>` and `-o wide` mean: output data with more details
```
# all resources
kubectl get all -o wide

# pods
kubectl get pods -o wide

# deployment
kubectl get deployments -o wide

# services
kubectl get services -o wide

```

## Log pod

use `kubectl get pods` to get name of pods first

```
kubectl logs -f --tail=200 <pod_name>

```

## Exec into pod's terminal

```
kubectl exec --stdin --tty <name-of-pod> -- /bin/bash

```


## Scale pods

Use `kubectl get deployments -o wide` to get deployment name
To turn off a running application, simply scale the deployment to **0**

```
kubectl scale deployment <deployment-name> --replicas=<number_of_replicas>
```

## Delete pod

This command will delete a pod. If deployment scale is larger than 0, deleted pod will be replace by the new one

```
kubectl delete pod <pod-name>
```

## Restart core dns when error

If application cannot resolve the mapping service name, you should restart the k8s's coredns

```
kubectl -n kube-system rollout restart deployment coredns
```
