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

## Namespace stuck at Terminating 

Firstly export your namespace name in env which got struck in Terminating state

```
export NAMESPACE=<namespace>
```
Then run below command to delete the Terminating namespace

```
kubectl get namespace $NAMESPACE -o json   | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/"   | kubectl replace --raw /api/v1/namespaces/$NAMESPACE/finalize -f -
```
# ingress-nginx-controller-admission-not-found 

Fix: [ingress-nginx-controller-admission-not-found](https://stackoverflow.com/questions/61365202/nginx-ingress-service-ingress-nginx-controller-admission-not-found)

```
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```
