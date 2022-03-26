# 03-canary

## Introduction

Let's run a canary release using just basic primitive resources: Deployments, Services and Ingress rules.

The main idea is to deploy two webapp versions and use the same selector for both deployments, therefore both versions will be availabe through the same service. The new version deployment should start with the minimun replicas, once the new version is checked, one can increase the number of replicas and reduce the initial deployment replicas.

The benefit of this approach is that one can check how behaves new versions affecting just a reduced number of users, and progress the update or rollback it depending the results.

## Deployment

Let's deploy the initial webapp version with its respective service and ingress rule:

```bash
$ kubectl apply --filename 01-canary-v1.yaml
deployment.apps/webapp created
service/webapp created
ingress.networking.k8s.io/webapp created
```

Access to the webapp though a browser and check its version:

```bash
$ google-chrome http://webapp.localhost
Opening in existing browser session.
```

Now let's deploy the new webapp version in a new deployment reusing the same selector:

```bash
$ kubectl apply --filename 02-canary-v2.yaml
deployment.apps/webapp-canary created
```

See how there are two deployments with the same selector used by the service, the initial version deployment has all its replicas, while the new one just one replica:

```bash
$ kubectl get deployments,service -o wide
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS     IMAGES                      SELECTOR
deployment.apps/webapp          3/3     3            3           5m8s   webapp-color   kodekloud/webapp-color:v1   app=webapp
deployment.apps/webapp-canary   1/1     1            1           116s   webapp-color   kodekloud/webapp-color:v2   app=webapp

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE    SELECTOR
service/webapp   ClusterIP   10.43.155.106   <none>        8080/TCP   5m8s   app=webapp
```

So at this point, 1/4 of the webapp calls shows the new version:

```bash
$ google-chrome http://webapp.localhost
Opening in existing browser session.
```

Now its time to check the new webapp version healthness, if everything looks fine with the new version, one can scale it out while scaling in to zero the initial deployment:

```bash
$ kubectl scale deployment webapp-canary --replicas 3
deployment.apps/webapp-canary scaled

$ kubectl scale deployment webapp --replicas 0
deployment.apps/webapp scaled

$ kubectl get deployments -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS     IMAGES                      SELECTOR
webapp-canary   3/3     3            3           13m   webapp-color   kodekloud/webapp-color:v2   app=webapp
webapp          0/0     0            0           16m   webapp-color   kodekloud/webapp-color:v1   app=webapp
```

Now all webapp calls should show the new version:

```bash
$ google-chrome http://webapp.localhost
Opening in existing browser session.
```

The canary release is done, and we can remove the old deployment, or keep it to perform a rollback just in case.

## Rollback

If something fails during the sanity checks, the canary release could be rolled back to its previous version, and just a minimun set of users would be affected.

```bash
$ kubectl scale deployment webapp-canary --replicas 0
deployment.apps/webapp-canary scaled

$ kubectl get deployments -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS     IMAGES                      SELECTOR
webapp          3/3     3            3           28m   webapp-color   kodekloud/webapp-color:v1   app=webapp
webapp-canary   0/0     0            0           25m   webapp-color   kodekloud/webapp-color:v2   app=webapp

```

## Clean up

Before going to the next scenario clean up the namespace:

```bash
$ kubectl delete --filename 01-canary-v1.yaml
deployment.apps/webapp deleted
service/webapp deleted
ingress.networking.k8s.io/webapp deleted

$ kubectl delete --filename 02-canary-v2.yaml
deployment.apps/webapp-canary deleted
```
