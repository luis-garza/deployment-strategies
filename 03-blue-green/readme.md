# 03-blue-green

## Introduction

Let's run a blue-green deployment using just basic primitive resources: Deployments, Services and Ingress rules.

The main idea is recreate two full environments of the webapp, one with the current or live version, another with the next or staging version, and switch the traffic between the two environments.

The great benefit of this approach is that one can run isolated tests the next version without compromising the live service, but in the other hand, it doubles teh required resources and it complexity increases exponentially replicating multiple services or components.

## Deployment

Let's deploy two webapp versions with their respective service, and create an ingress rule to access to both of versions:

| Ingress rule | Service | Deployment | Version |
|--------------|---------|------------|---------|
| staging.localhost | blue service | blue deployment | v1 |
| live.localhost | green service | green deployment | v2 |

```bash
$ kubectl apply --filename 01-blue-green-v1.yaml
deployment.apps/webapp-blue created
service/webapp-blue created

$ kubectl apply --filename 02-blue-green-v2.yaml
deployment.apps/webapp-green created
service/webapp-green created

$ kubectl apply --filename 03-blue-green-ingress.yaml
ingress.networking.k8s.io/webapp created
```

Check how looks like the resources:

```bash
$ kubectl get deployments.apps --output wide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
webapp-blue    3/3     3            3           2m26s   webapp-color   kodekloud/webapp-color:v1   app=webapp,status=blue
webapp-green   3/3     3            3           2m6s    webapp-color   kodekloud/webapp-color:v2   app=webapp,status=green

$ kubectl describe ingress webapp
Name:             webapp
Labels:           <none>
Namespace:        demo
Address:          172.18.0.2,172.18.0.4,172.18.0.5
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path  Backends
  ----               ----  --------
  staging.localhost
                     /   webapp-blue:http (10.42.1.15:8080,10.42.2.14:8080,10.42.3.11:8080)
  live.localhost
                     /   webapp-green:http (10.42.1.16:8080,10.42.2.15:8080,10.42.3.12:8080)
Annotations:         <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    50s (x2 over 86s)  nginx-ingress-controller  Scheduled for sync
```

Now open and check both webapp versions though a browser:

```bash
$ google-chrome http://live.localhost
Opening in existing browser session.

$ google-chrome http://staging.localhost
Opening in existing browser session.
```

Now let's update the staging environment with the last version.

```yaml
spec:
  template:
    spec:
      containers:
        - name: webapp-color
          image: kodekloud/webapp-color:v3
```

To do it so, let's update deployment which is representing the staging environment, the blue one in this case:

```bash
$ kubectl patch deploy webapp-blue --patch-file 04-blue-green-v3-patch.yaml
deployment.apps/webapp-blue patched

$ kubectl get deployments.apps --output wide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS     IMAGES                      SELECTOR
webapp-green   3/3     3            3           4m10s   webapp-color   kodekloud/webapp-color:v2   app=webapp,status=green
webapp-blue    3/3     3            3           4m30s   webapp-color   kodekloud/webapp-color:v3   app=webapp,status=blue
```

At this point the new version is fully accessible through the staging environment, it could be fully tested and checked while the current version is on live with no affectation.

| Ingress rule | Service | Deployment | Version |
|--------------|---------|------------|---------|
| staging.localhost | blue service | blue deployment | v3 |
| live.localhost | green service | green deployment | v2 |

Figure the staging tests looks fine, let's move the latest version to live! It's as simple as interchange the target services in both live & staging ingress rules:

```bash
$ kubectl apply --filename 05-blue-green-ingress-patch.yaml
ingress.networking.k8s.io/webapp configured

$ kubectl describe ingress webapp
Name:             webapp
Labels:           <none>
Namespace:        demo
Address:          172.18.0.2,172.18.0.4,172.18.0.5
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path  Backends
  ----               ----  --------
  staging.localhost
                     /   webapp-green:http (10.42.1.16:8080,10.42.2.15:8080,10.42.3.12:8080)
  live.localhost
                     /   webapp-blue:http (10.42.1.17:8080,10.42.2.16:8080,10.42.3.13:8080)
Annotations:         <none>
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    50s (x3 over 4m50s)  nginx-ingress-controller  Scheduled for sync
```

| Ingress rule | Service | Deployment | Version |
|--------------|---------|------------|---------|
| live.localhost | blue service | blue deployment | v3 |
| staging.localhost | green service | green deployment | v2 |

Note that in this example the traffic switch is done through ingress rules, but it could be handled updating the services to point to the desired workload.

To save resources once can scale in or remove the staging environment, the green deployment in this case.

## Rollback

You can imagine how easy is to perform a rollback, one only have to recover the previous ingress rules state, immediately the previous version is available through the live traffic.

```bash
$ kubectl apply --filename 03-blue-green-ingress.yaml
ingress.networking.k8s.io/webapp configured

$ kubectl describe ingress webapp
Name:             webapp
Labels:           <none>
Namespace:        demo
Address:          172.18.0.2,172.18.0.4,172.18.0.5
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path  Backends
  ----               ----  --------
  staging.localhost
                     /   webapp-blue:http (10.42.1.17:8080,10.42.2.16:8080,10.42.3.13:8080)
  live.localhost
                     /   webapp-green:http (10.42.1.16:8080,10.42.2.15:8080,10.42.3.12:8080)
Annotations:         <none>
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    13s (x4 over 7m37s)  nginx-ingress-controller  Scheduled for sync
```

| Ingress rule | Service | Deployment | Version |
|--------------|---------|------------|---------|
| staging.localhost | blue service | blue deployment | v3 |
| live.localhost | green service | green deployment | v2 |

## Clean up

Before going to the next scenario clean up the namespace:

```bash
$ kubectl delete --filename 01-blue-green-v1.yaml
deployment.apps "webapp-blue" deleted
service "webapp-blue" deleted

$ kubectl delete --filename 02-blue-green-v2.yaml
deployment.apps "webapp-green" deleted
service "webapp-green" deleted

$ kubectl delete --filename 03-blue-green-ingress.yaml
ingress.networking.k8s.io "webapp" deleted
```
