# 03-argo-rollout

## Introduction

The [Argo project](https://argoproj.github.io) provides a CRD which replaces the deployment resources to handle blue/green and canary releases like a pro.

It simplifies the whole process, and enables automated decisions based on observability metrics like Prometheus, and it's highly integrated with ingress controllers such nginx or service meshes like Istio.

Let's scrap in a small and siple canary release.

## Installation

First let's install the controller and CRDs following the [official documentation](https://argoproj.github.io/argo-rollouts/installation/):

```bash
$ kubectl create namespace argo-rollouts
namespace/argo-rollouts created

$ kubectl apply --namespace argo-rollouts --filename https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
customresourcedefinition.apiextensions.k8s.io/analysisruns.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/analysistemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/clusteranalysistemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/experiments.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/rollouts.argoproj.io created
serviceaccount/argo-rollouts created
clusterrole.rbac.authorization.k8s.io/argo-rollouts created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-view created
clusterrolebinding.rbac.authorization.k8s.io/argo-rollouts created
secret/argo-rollouts-notification-secret created
service/argo-rollouts-metrics created
deployment.apps/argo-rollouts created
```

Also install a [CLI plugin](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation) to enhance the user experience of Argo rollouts, note that next instructions are for Linux:

```bash
$ sudo curl --location --silent --output /usr/local/bin/kubectl-argo-rollouts https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64

$ sudo chmod ugo+rx /usr/local/bin/kubectl-argo-rollouts

$ kubectl-argo-rollouts version
kubectl-argo-rollouts: v1.2.0-rc2+7b69058
  BuildDate: 2022-02-24T22:32:47Z
  GitCommit: 7b690580f69fd491f40c15b3ff4031f13bdae332
  GitTreeState: clean
  GoVersion: go1.17.6
  Compiler: gc
  Platform: linux/amd64
```

## Deployment

Let's deploy the initial webapp version with its respective service and ingress rule, but using a rollout instead a deployment resource. See how the `steps` list defines the canary release strategy:

```bash
$ kubectl apply --filename 01-argo-rollgout-v1.yaml
NAME                          READY   STATUS    RESTARTS   AGE     IP          NODE              NOMINATED NODE   READINESS GATES
pod/webapp-579884787f-9hth6   1/1     Running   0          2m15s   10.42.2.9   k3d-lab-agent-2   <none>           <none>
pod/webapp-579884787f-vkgvt   1/1     Running   0          2m15s   10.42.2.8   k3d-lab-agent-2   <none>           <none>
pod/webapp-579884787f-27vw8   1/1     Running   0          2m15s   10.42.3.6   k3d-lab-agent-1   <none>           <none>
pod/webapp-579884787f-bdrh7   1/1     Running   0          2m15s   10.42.1.7   k3d-lab-agent-0   <none>           <none>

$ kubectl kubectl get all,ingress,rollout -o wide
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/webapp   ClusterIP   10.43.150.136   <none>        8080/TCP   2m15s   app=webapp

NAME                                DESIRED   CURRENT   READY   AGE     CONTAINERS     IMAGES                      SELECTOR
replicaset.apps/webapp-579884787f   4         4         4       2m15s   webapp-color   kodekloud/webapp-color:v1   app=webapp,rollouts-pod-template-hash=579884787f

NAME                               CLASS   HOSTS              ADDRESS                            PORTS   AGE
ingress.networking.k8s.io/webapp   nginx   webapp.localhost   172.18.0.2,172.18.0.4,172.18.0.5   80      2m15s

NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rollout.argoproj.io/webapp   4         4         4            4           2m15s
```

Now open the webapp in the browser, and check how looks like the rollout:

```bash
$ google-chrome http://webapp.localhost
Opening in existing browser session.

$ kubectl describe rollouts webapp
Name:         webapp
Namespace:    demo
Labels:       <none>
Annotations:  rollout.argoproj.io/revision: 1
API Version:  argoproj.io/v1alpha1
Kind:         Rollout
Metadata:
  Creation Timestamp:  2022-03-15T22:23:11Z
  Generation:          2
  Resource Version:  4768
  UID:               e258bda2-8676-4ea6-9293-f9a3d21a581c
Spec:
  Replicas:  4
  Selector:
    Match Labels:
      App:  webapp
  Strategy:
    Canary:
      Steps:
        Set Weight:  25
        Pause:
        Set Weight:  50
        Pause:
          Duration:  3m
        Set Weight:  75
        Pause:
          Duration:  3m
        Set Weight:  100
  Template:
    Metadata:
      Labels:
        App:  webapp
    Spec:
      Containers:
        Image:  kodekloud/webapp-color:v1
        Name:   webapp-color
        Ports:
          Container Port:  8080
          Name:            http
          Protocol:        TCP
        Resources:
Status:
  HPA Replicas:        4
  Available Replicas:  4
  Blue Green:
  Canary:
  Conditions:
    Last Transition Time:  2022-03-15T22:23:13Z
    Last Update Time:      2022-03-15T22:23:13Z
    Message:               RolloutCompleted
    Reason:                RolloutCompleted
    Status:                True
    Type:                  Completed
    Last Transition Time:  2022-03-15T22:23:11Z
    Last Update Time:      2022-03-15T22:23:13Z
    Message:               ReplicaSet "webapp-579884787f" has successfully progressed.
    Reason:                NewReplicaSetAvailable
    Status:                True
    Type:                  Progressing
    Last Transition Time:  2022-03-15T22:23:13Z
    Last Update Time:      2022-03-15T22:23:13Z
    Message:               Rollout has minimum availability
    Reason:                AvailableReason
    Status:                True
    Type:                  Available
  Current Pod Hash:        579884787f
  Current Step Hash:       6d8c56cb4
  Current Step Index:      7
  Observed Generation:     2
  Phase:                   Healthy
  Ready Replicas:          4
  Replicas:                4
  Selector:                app=webapp
  Stable RS:               579884787f
  Updated Replicas:        4
Events:
  Type    Reason                Age    From                 Message
  ----    ------                ----   ----                 -------
  Normal  RolloutUpdated        3m56s  rollouts-controller  Rollout updated to revision 1
  Normal  NewReplicaSetCreated  3m56s  rollouts-controller  Created ReplicaSet webapp-579884787f (revision 1)
  Normal  ScalingReplicaSet     3m56s  rollouts-controller  Scaled up ReplicaSet webapp-579884787f (revision 1) from 0 to 4
  Normal  RolloutCompleted      3m56s  rollouts-controller  Rollout completed update to revision 1 (579884787f): Initial deploy
```

As you can see there is lot of detailed information. The CLI plugin describes better the rollout, and at the same time it enables new commands.

```bash
$ kubectl-argo-rollouts get rollout  webapp

Name:            webapp
Namespace:       demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          7/7
  SetWeight:     100
  ActualWeight:  100
Images:          kodekloud/webapp-color:v1 (stable)
Replicas:
  Desired:       4
  Current:       4
  Updated:       4
  Ready:         4
  Available:     4

NAME                                KIND        STATUS     AGE    INFO
⟳ webapp                            Rollout     ✔ Healthy  9m23s
└──# revision:1
   └──⧉ webapp-579884787f           ReplicaSet  ✔ Healthy  9m23s  stable
      ├──□ webapp-579884787f-27vw8  Pod         ✔ Running  9m23s  ready:1/1
      ├──□ webapp-579884787f-9hth6  Pod         ✔ Running  9m23s  ready:1/1
      ├──□ webapp-579884787f-bdrh7  Pod         ✔ Running  9m23s  ready:1/1
      └──□ webapp-579884787f-vkgvt  Pod         ✔ Running  9m23s  ready:1/1
```

The CLI plugin provides a dashboard to visualize and operate the rollouts, open it in a new shell:

```bash
$ kubectl-argo-rollouts dashboard
INFO[0000] Argo Rollouts Dashboard is now available at localhost 3100

$ google-chrome http://localhost:3100
Opening in existing browser session.
```

Now let's update the webapp version, and therefore trigger a canary release:

```bash
$ kubectl patch rollout webapp --type merge --patch-file 02-argo-rollgout-v2-patch.yaml
rollout.argoproj.io/webapp patched
```

Check how looks like the rollout in both CLI and dashboard, and see how the replicaset are weight and the rollout is paused waiting for an action:

```bash
$ kubectl-argo-rollouts get rollout webapps

Name:            webapp
Namespace:       demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/7
  SetWeight:     25
  ActualWeight:  25
Images:          kodekloud/webapp-color:v1 (stable)
                 kodekloud/webapp-color:v2 (canary)
Replicas:
  Desired:       4
  Current:       4
  Updated:       1
  Ready:         4
  Available:     4

NAME                                KIND        STATUS     AGE  INFO
⟳ webapp                            Rollout     ॥ Paused   20m
├──# revision:2
│  └──⧉ webapp-85f676769b           ReplicaSet  ✔ Healthy  69s  canary
│     └──□ webapp-85f676769b-rwwqp  Pod         ✔ Running  69s  ready:1/1
└──# revision:1
   └──⧉ webapp-579884787f           ReplicaSet  ✔ Healthy  20m  stable
      ├──□ webapp-579884787f-27vw8  Pod         ✔ Running  20m  ready:1/1
      ├──□ webapp-579884787f-bdrh7  Pod         ✔ Running  20m  ready:1/1
      └──□ webapp-579884787f-vkgvt  Pod         ✔ Running  20m  ready:1/1
```

At this point, depending on the success of the new version, one can promote the rollout for the next step, or abort the rollout to recover the previous state.

```bash
$ kubectl-argo-rollouts promote webapp
rollout 'webapp' promoted

$ kubectl-argo-rollouts get rollout webapp

Name:            webapp
Namespace:       demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/7
  SetWeight:     50
  ActualWeight:  50
Images:          kodekloud/webapp-color:v1 (stable)
                 kodekloud/webapp-color:v2 (canary)
Replicas:
  Desired:       4
  Current:       4
  Updated:       2
  Ready:         4
  Available:     4

NAME                                KIND        STATUS         AGE    INFO
⟳ webapp                            Rollout     ॥ Paused       21m
├──# revision:2
│  └──⧉ webapp-85f676769b           ReplicaSet  ✔ Healthy      2m36s  canary
│     ├──□ webapp-85f676769b-rwwqp  Pod         ✔ Running      2m36s  ready:1/1
│     └──□ webapp-85f676769b-f2gr2  Pod         ✔ Running      9s     ready:1/1
└──# revision:1
   └──⧉ webapp-579884787f           ReplicaSet  ✔ Healthy      21m    stable
      ├──□ webapp-579884787f-27vw8  Pod         ✔ Running      21m    ready:1/1
      ├──□ webapp-579884787f-bdrh7  Pod         ◌ Terminating  21m    ready:1/1
      └──□ webapp-579884787f-vkgvt  Pod         ✔ Running      21m    ready:1/1
```

As the following canary release steps are automated defined by a timer, the rollout will increase the weight of the new version accordingly until the rollout finish.

## Rollback

The rollout rollbacks could be triggered automatically based on application healthness and metrics. In this case let's rollback the rollout manually.

First let's see how the rollback is healthy and completed:

```bash
$ kubectl-argo-rollouts get rollout webapp

Name:            webapp
Namespace:       demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          7/7
  SetWeight:     100
  ActualWeight:  100
Images:          kodekloud/webapp-color:v2 (stable)
Replicas:
  Desired:       4
  Current:       4
  Updated:       4
  Ready:         4
  Available:     4

NAME                                KIND        STATUS        AGE    INFO
⟳ webapp                            Rollout     ✔ Healthy     28m
├──# revision:2
│  └──⧉ webapp-85f676769b           ReplicaSet  ✔ Healthy     9m25s  stable
│     ├──□ webapp-85f676769b-rwwqp  Pod         ✔ Running     9m25s  ready:1/1
│     ├──□ webapp-85f676769b-f2gr2  Pod         ✔ Running     6m58s  ready:1/1
│     ├──□ webapp-85f676769b-cs9n2  Pod         ✔ Running     3m54s  ready:1/1
│     └──□ webapp-85f676769b-v4xsx  Pod         ✔ Running     50s    ready:1/1
└──# revision:1
   └──⧉ webapp-579884787f           ReplicaSet  • ScaledDown  28m
```

Figure an error has been found in the latest version, let's recover the initial web app version:

```bash
$ kubectl-argo-rollouts undo webapp
rollout 'webapp' undo

$ kubectl-argo-rollouts get rollout webapp

Name:            webapp
Namespace:       demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/7
  SetWeight:     25
  ActualWeight:  25
Images:          kodekloud/webapp-color:v1 (canary)
                 kodekloud/webapp-color:v2 (stable)
Replicas:
  Desired:       4
  Current:       4
  Updated:       1
  Ready:         4
  Available:     4

NAME                                KIND        STATUS     AGE    INFO
⟳ webapp                            Rollout     ॥ Paused   31m
├──# revision:3
│  └──⧉ webapp-579884787f           ReplicaSet  ✔ Healthy  31m    canary
│     └──□ webapp-579884787f-zxfkr  Pod         ✔ Running  74s    ready:1/1
└──# revision:2
   └──⧉ webapp-85f676769b           ReplicaSet  ✔ Healthy  12m    stable
      ├──□ webapp-85f676769b-rwwqp  Pod         ✔ Running  12m    ready:1/1
      ├──□ webapp-85f676769b-f2gr2  Pod         ✔ Running  10m    ready:1/1
      └──□ webapp-85f676769b-cs9n2  Pod         ✔ Running  7m10s  ready:1/1
```

Recall it's an emergency, so in this case we won't to wait between the release steps:

```bash
$ kubectl-argo-rollouts promote webapp --full
rollout 'webapp' fully promoted
```

## Clean up

Before close the workshop clean up the namespace:

```bash
$ kubectl delete --filename 01-argo-rollgout-v1.yaml
rollout.argoproj.io "webapp" deleted
service "webapp" deleted
ingress.networking.k8s.io "webapp" deleted
```

And additionally delete the local Kubernetes cluster:

```bash
$ k3d cluster delete lab
INFO[0000] Deleting cluster 'lab'
INFO[0003] Deleting cluster network 'k3d-lab'
INFO[0003] Deleting image volume 'k3d-lab-images'
INFO[0004] Removing cluster details from default kubeconfig...
INFO[0004] Removing standalone kubeconfig file (if there is one)...
INFO[0004] Successfully deleted cluster lab!
```
