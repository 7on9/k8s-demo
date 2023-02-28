# Start using your cluster 

## Install NGINX

The following steps below will show how to setup nginx ingress controller for bare metal cluster. For other type machine, see: [Setup NGINX Ingress controller](./NGINX_INGRESS_CONTROLLER.md)

1. Get nginx ingress deployment configuration file: 


```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/baremetal/deploy.yaml
```

This command will download file `deploy.yaml` to your folder

Edit `deploy.yaml`

- Deployment: Add `hostNetwork` to `spec`

```
kind: Deployment
spec:
  template:
    spec:
      hostNetwork: true
```

- Services: Add `externalTrafficPolicy: Cluster`
```
apiVersion: v1
kind: Service
metadata:
  ...
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Cluster
```

2. Apply configuration file:

```
kubectl apply -f deploy.yaml
```

3. Check your k8s:

```
kubectl get all -o wide -n ingress-nginx
```

The result should look similar to this: 

```
NAME                                            READY   STATUS      RESTARTS      AGE     IP              NODE       NOMINATED NODE   READINESS GATES
pod/ingress-nginx-admission-create-zkj78        0/1     Completed   0             3h24m   10.244.0.90     worker-1   <none>           <none>
pod/ingress-nginx-admission-patch-fc9k6         0/1     Completed   0             3h24m   10.244.0.89     worker-1   <none>           <none>
pod/ingress-nginx-controller-6f6f56f4b4-tzfcc   1/1     Running     0             171m    172.104.39.12   worker-1   <none>           <none>

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
service/ingress-nginx-controller             NodePort    10.98.200.238   <none>        80:30823/TCP,443:32693/TCP   3h24m   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
service/ingress-nginx-controller-admission   ClusterIP   10.96.192.123   <none>        443/TCP                      3h24m   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS      IMAGES                                                                                                                    SELECTOR
deployment.apps/ingress-nginx-controller   1/1     1            1           3h24m   controller      registry.k8s.io/ingress-nginx/controller:v1.6.4@sha256:15be4666c53052484dd2992efacf2f50ea77a78ae8aa21ccd91af6baaa7ea22f   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx

NAME                                                  DESIRED   CURRENT   READY   AGE     CONTAINERS      IMAGES                                                                                                                    SELECTOR
replicaset.apps/ingress-nginx-controller-54d878c797   0         0         0       3h24m   controller      registry.k8s.io/ingress-nginx/controller:v1.6.4@sha256:15be4666c53052484dd2992efacf2f50ea77a78ae8aa21ccd91af6baaa7ea22f   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx,pod-template-hash=54d878c797
replicaset.apps/ingress-nginx-controller-6f6f56f4b4   1         1         1       171m    controller      registry.k8s.io/ingress-nginx/controller:v1.6.4@sha256:15be4666c53052484dd2992efacf2f50ea77a78ae8aa21ccd91af6baaa7ea22f   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx,pod-template-hash=6f6f56f4b4

NAME                                       COMPLETIONS   DURATION   AGE     CONTAINERS   IMAGES                                                                                                                                            SELECTOR
job.batch/ingress-nginx-admission-create   1/1           4s         3h24m   create       registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f   controller-uid=d12fcb6f-4630-49a5-9983-1d4adfcc43c5
job.batch/ingress-nginx-admission-patch    1/1           4s         3h24m   patch        registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f   controller-uid=3f23465c-d1a7-4bbb-a0cd-536ad7c68cbb
```

If your Kubernetes cluster is a "real" cluster that supports services of type LoadBalancer, it will have allocated an external IP address or FQDN to the ingress controller.

You can see that IP address or FQDN with the following command:

```
kubectl get service ingress-nginx-controller --namespace=ingress-nginx
```

It will be the EXTERNAL-IP field. If that field shows <pending>, this means that your Kubernetes cluster wasn't able to provision the load balancer (generally, this is because it doesn't support services of type LoadBalancer). To fix this do the following steps:

Patch External IP 


```
kubectl patch svc ingress-nginx-controller -p '{"spec": {"type": "LoadBalancer", "externalIPs":["<Your_public_IP>"]}}' --namespace=ingress-nginx
```

4. Create ingress to public your cluster

Create file `ingress-nginx.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-v1
  namespace: ingress-nginx
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "your.domain"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: ingress-nginx-controller
            port:
              number: 80
```

Apply ingress:

```
kubectl apply -f ingress-nginx.yaml
```

Error can show up: 

```
Error from server (InternalError): error when creating "ingress.yaml": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout=10s": dial tcp 10.108.75.196:443: i/o timeout
```

Resolve it by run this command:

```
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```

Run apply ingress again.

5. Public your application:

- Deploy new application or using sample project [demo-express-mongo](../demo-express-mongo/) (edit namespace match with ingress nginx)

- Create file ingress, with sample project I create new file `mongo-ingress.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: ingress-nginx
  name: ingress-mongo-express
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: your.mongo.express.domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mongo-express-service
            port:
              number: 8081
```

- Apply ingress: 

```
kubectl apply -f mongo-ingress.yaml
```

## Install Trafik
