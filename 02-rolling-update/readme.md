# 02-rolling-update

## Introduction

A part of recreate there is a more advanced strategy type, RollingUpdate. It schedules new pods in parallel with the existing ones, and stops the old pods leaving the new ones in a controlled way. It's the default strategy type of Deployments.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

With this strategy type there is no downtime during an update, but there is a short period of time where both versions run at the same time.

The amount of pods unavailable or exceeding the replica limit could be defined through `maxUnavailable` and `maxSurge` fields.

## Deployment

Deploy a basic webapp deployment, exposing the app though a service and an ingress rule, note that one should update the ingress host domain according to the Kubernetes lab domain:

```bash
$ kubectl apply --filename 01-rolling-update-v1.yaml
deployment.apps/webapp created
service/webapp created
ingress.networking.k8s.io/webapp created
```

Check what has been deployed: the deployment, a replicaset handling the first version, the 5 pod replicas, and its service and ingress:

```bash
$ kubectl get all,ingress --output wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-gr5rf   1/1     Running   0          31s   10.42.1.10   k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-hvhwv   1/1     Running   0          31s   10.42.1.9    k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-6rh7g   1/1     Running   0          31s   10.42.2.9    k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-s876t   1/1     Running   0          31s   10.42.2.8    k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-5vdtv   1/1     Running   0          31s   10.42.3.8    k3d-lab-agent-2   <none>           <none>

NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/webapp   ClusterIP   10.43.177.91   <none>        8080/TCP   31s   app=webapp

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   5/5     5            0           31s   webapp-color   kodekloud/webapp-color:v1   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE   CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   5         5         5       31s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                               CLASS   HOSTS              ADDRESS   PORTS   AGE
ingress.networking.k8s.io/webapp   nginx   webapp.localhost             80      31s
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
deployment.apps/webapp   5/5     5            5           3m9s   webapp-color   kodekloud/webapp-color:v1   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE    CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   5         5         5       3m9s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                          READY   STATUS    RESTARTS   AGE    IP           NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-gr5rf   1/1     Running   0          3m9s   10.42.1.10   k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-hvhwv   1/1     Running   0          3m9s   10.42.1.9    k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-6rh7g   1/1     Running   0          3m9s   10.42.2.9    k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-s876t   1/1     Running   0          3m9s   10.42.2.8    k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-5vdtv   1/1     Running   0          3m9s   10.42.3.8    k3d-lab-agent-2   <none>           <none>
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
$ kubectl patch deploy webapp --patch-file 02-rolling-update-v2-patch.yaml
deployment.apps/webapp patched
```

Check how the webapp behaves in the browser, and how a new replicaset has been created referring to the new updated image.
In this point the webapp service points to both versions, as the service selector `app=webapp` matches with both replicaset selectors.

The pods roll out starts respecting the rules specified by the `maxUnavailable` and `maxSurge` fields, terminating pods from the initial replicaset, and starting new pods for the newer replicaset.

```bash
$ kubectl get deployments,replicasets,pod --output wide
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   5/5     2            5           4m11s   webapp-color   kodekloud/webapp-color:v2   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-78bf8458d6   4         4         4       4m11s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6
replicaset.apps/webapp-8545755c47   2         2         1       31s     webapp-color   kodekloud/webapp-color:v2   app=webapp,pod-template-hash=8545755c47

NAME                          READY   STATUS              RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
pod/webapp-78bf8458d6-gr5rf   1/1     Running             0          4m11s   10.42.1.10   k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-hvhwv   1/1     Running             0          4m11s   10.42.1.9    k3d-lab-agent-1   <none>           <none>
pod/webapp-78bf8458d6-6rh7g   1/1     Running             0          4m11s   10.42.2.9    k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-5vdtv   1/1     Running             0          4m11s   10.42.3.8    k3d-lab-agent-2   <none>           <none>
pod/webapp-8545755c47-px7k6   1/1     Running             0          31s     10.42.2.10   k3d-lab-agent-0   <none>           <none>
pod/webapp-78bf8458d6-s876t   1/1     Terminating         0          4m11s   10.42.2.8    k3d-lab-agent-0   <none>           <none>
pod/webapp-8545755c47-5kftc   0/1     ContainerCreating   0          1s      <none>       k3d-lab-agent-1   <none>           <none>
```

Give it time, refresh the webapp in the browser several times to see how tboth versions are running at the same time. Finally, once the deployment is done, the newer replicaset has all its pods running with the updated image version, while the initial replicaset has no pods.

```bash
$ kubectl get deployments,replicasets,pod --output wide
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp   5/5     5            5           6m56s   webapp-color   kodekloud/webapp-color:v2   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-8545755c47   5         5         5       3m16s   webapp-color   kodekloud/webapp-color:v2   app=webapp,pod-template-hash=8545755c47
replicaset.apps/webapp-78bf8458d6   0         0         0       6m56s   webapp-color   kodekloud/webapp-color:v1   app=webapp,pod-template-hash=78bf8458d6

NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
pod/webapp-8545755c47-px7k6   1/1     Running   0          3m16s   10.42.2.10   k3d-lab-agent-0   <none>           <none>
pod/webapp-8545755c47-5kftc   1/1     Running   0          2m46s   10.42.1.11   k3d-lab-agent-1   <none>           <none>
pod/webapp-8545755c47-7c25b   1/1     Running   0          2m15s   10.42.3.9    k3d-lab-agent-2   <none>           <none>
pod/webapp-8545755c47-7gltt   1/1     Running   0          103s    10.42.2.11   k3d-lab-agent-0   <none>           <none>
pod/webapp-8545755c47-4lcc8   1/1     Running   0          72s     10.42.1.12   k3d-lab-agent-1   <none>           <none>
```

## Rollback

In terms of rollback, it doesn't differs from the Recreate deployment strategy, it relays on previous replicasets. The only difference between both strategies is how the pods are replaced.

```bash
$ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

$ kubectl rollout undo deployment webapp
deployment.apps/webapp rolled back

$ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
```

## Clean up

Before going further let's clean up the namespace:

```bash
$ kubectl delete --filename 01-rolling-update-v1.yaml
deployment.apps "webapp" deleted
service "webapp" deleted
ingress.networking.k8s.io "webapp" deleted
```
