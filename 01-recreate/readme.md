# 01-recreate

## Introduction

Recreate is the simplest deployment strategy to perform an update, it stops the existing pods, and only when all are stopped then the new ones will be started.

```yaml
spec:
  strategy:
    type: Recreate
```

This method causes downtime during the update, as there will be a short period of time where there won't be any pod ready. Despite the downtime there are use cases where it makes sense, for instance in scenarios where an application can not work with different versions at the same time to avoid consintency issues.

## Deployment

Deploy a basic webapp deployment, exposing it though a service and an ingress rule, note that one should update the ingress host domain according to the Kubernetes lab domain:

```bash
$ kubectl apply --filename 01-recreate-v1.yaml
deployment.apps/webapp created
service/webapp created
ingress.networking.k8s.io/webapp created
```

Check what has been deployed: the deployment, a replicaset handling the first version, the 3 pod replicas, and its service and ingress:

```bash
$ kubectl get all,ingress --output wide
NAME                          READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-5nldz   1/1     Running   0          16s   10.42.2.5   k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-tgcnw   1/1     Running   0          16s   10.42.3.5   k3d-lab-agent-2   <none>           <none>
pod/webapp-78bf8458d6-7266b   1/1     Running   0          16s   10.42.1.6   k3d-lab-agent-1   <none>           <none>

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/webapp   ClusterIP   10.43.228.171   <none>        8080/TCP   16s   app=webapp

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   3/3     3            3           16s   webapp-color   kodekloud/webapp-color:v1   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE   CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   3         3         3       16s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                               CLASS   HOSTS              ADDRESS   PORTS   AGE
ingress.networking.k8s.io/webapp   nginx   webapp.localhost             80      16s
```

Access to the webapp though a browser and check its version:

```bash
$ google-chrome http://webapp.localhost
Opening in existing browser session.
```

Open another terminal in leave it watching the deployed resources:

```bash
$ watch --interval 1 kubectl get deployments,replicasets,pod --output wide
Every 1.0s: kubectl get deployments,replicasets,pod --output wide

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   3/3     3            3           2m6s   webapp-color   kodekloud/webapp-color:v1   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE    CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   3         3         3       2m6s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                          READY   STATUS    RESTARTS   AGE    IP          NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-5nldz   1/1     Running   0          2m6s   10.42.2.5   k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-tgcnw   1/1     Running   0          2m6s   10.42.3.5   k3d-lab-agent-2   <none>           <none>
pod/webapp-78bf8458d6-7266b   1/1     Running   0          2m6s   10.42.1.6   k3d-lab-agent-1   <none>           <none>
```

Update the deployment with a newer image while keeping an eye to the terminal showing the resources:

```yaml
spec:
  template:
    spec:
      containers:
        - name: webapp-color
          image: kodekloud/webapp-color:v2
```

One can just edit the deployment, or patch it:

```bash
$ kubectl patch deploy webapp --patch-file 02-recreate-v2-patch.yaml
deployment.apps/webapp patched
```

Check how the webapp behaves in the browser, and how the replicaset has been scaled out to 0 replicas with all its pods stopped:

```bash
$ kubectl get deployments,replicasets,pod --output wide
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   0/3     0            0           5m39s   webapp-color   kodekloud/webapp-color:v2   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   0         0         0       5m39s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                          READY   STATUS        RESTARTS   AGE     IP          NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-7266b   1/1     Terminating   0          5m39s   10.42.1.6   k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-tgcnw   1/1     Terminating   0          5m39s   10.42.3.5   k3d-lab-agent-2   <none>           <none>
pod/webapp-78bf8458d6-5nldz   1/1     Terminating   0          5m39s   10.42.2.5   k3d-lab-agent-0   <none>           <none>
```

Later, refresh the webapp in the browser, and see how a new replicaset has been created with pods running the newer image version.

```bash
$ kubectl get deployments,replicasets,pod --output wide
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   3/3     3            3           6m36s   webapp-color   kodekloud/webapp-color:v2   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   0         0         0       6m36s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6
replicaset.apps/webapp-8545755c47   3         3         3       44s     webapp-color   kodekloud/webapp-color:v2   app=webapp,pod-template-hash=8545755c47

NAME                          READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
pod/webapp-8545755c47-xnvpc   1/1     Running   0          44s   10.42.2.6   k3d-lab-agent-0   <none>           <none>
pod/webapp-8545755c47-tc2r4   1/1     Running   0          44s   10.42.3.6   k3d-lab-agent-2   <none>           <none>
pod/webapp-8545755c47-hksbh   1/1     Running   0          44s   10.42.1.7   k3d-lab-agent-1   <none>           <none>
```

## Rollback

Newer versions of the deployments creates new replicasets while maintains the previous ones.

So a rollback should be as easy as scale out to 0 the latest replicaset, and then scale out the previous replicaset, like:

``` bash
$ kubectl scale replicaset webapp-8545755c47 --replicas 0
replicaset.apps/webapp-8545755c47 scaled

$ kubectl scale replicaset webapp-78bf8458d6 --replicas 3
replicaset.apps/webapp-78bf8458d6 scaled
```

Don't do it, this is not possible as the replicasets are being handled by the deployment resource, and the original values will be recovered.

To perform a rollback, one can just use the rollout command:

```bash
$ kubectl rollout status deployment webapp
deployment "webapp" successfully rolled out

$ kubectl get replicasets --output wide
NAME                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
webapp-78bf8458d6   0         0         0       8m9s    webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6
webapp-8545755c47   3         3         3       2m17s   webapp-color   kodekloud/webapp-color:v2   app=webapp,pod-template-hash=8545755c47

$ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

$ kubectl rollout undo deployment webapp
deployment.apps/webapp rolled back

$ kubectl get replicasets --output wide
NAME                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
webapp-8545755c47   0         0         0       3m19s   webapp-color   kodekloud/webapp-color:v2   app=webapp,pod-template-hash=8545755c47
webapp-78bf8458d6   3         3         0       9m11s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

$ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

Check in the browser how now is available the previous version again!

## Clean up

Before going to the next scenario clean up the namespace:

```bash
$ kubectl delete --filename 01-recreate-v1.yaml
deployment.apps "webapp" deleted
service "webapp" deleted
ingress.networking.k8s.io "webapp" deleted
```
