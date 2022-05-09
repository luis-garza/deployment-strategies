# 00-lab

## Introduction

There are some requirements in order to run the workshop exercises, the kubectl tool and a Kubernetes cluster.

The Kubernetes cluster should be up and running, with a ingress endpoint available. If you don't have access to any Kubernetes cluster, one can run it locally with [docker](https://www.docker.com) & [k3d](https://k3d.io/) help.

## kubectl

The kubectl tool is essential to work with Kubernetes, please install it and ensure it's available in your environment's path:

- [kubectl installation](https://kubernetes.io/docs/tasks/tools/#kubectl)

Next, check it's working:

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.6", GitCommit:"f59f5c2fda36e4036b49ec027e556a15456108f0", GitTreeState:"clean", BuildDate:"2022-01-19T17:33:06Z", GoVersion:"go1.16.12", Compiler:"gc", Platform:"linux/amd64"}
```

## Kubernetes

Skip this section if you already have access to a Kubernetes cluster.

### Dependencies

To deploy a local Kubernetes cluster one can use k3d tool, it simplifies the process using Docker containers, therefore one will need Docker engine installed:

- [Docker installation](https://docs.docker.com/get-docker)

Run `hello-world` container to check Docker engine is up and running:

```bash
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:97a379f4f88575512824f3b352bc03cd75e239179eea0fecc38e597b2209f49a
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Once Docker is set up in your environment, install k3d tool:

- [k3d installation](https://github.com/k3d-io/k3d/#get)

Finally check it's working:

```bash
$ k3d version
k3d version v5.2.2
k3s version v1.21.7-k3s1 (default)
```

### Local cluster

Let's deploy a local cluster, empowered by k3d capabilities we are going to deploy a multinode cluster with 1 server and 3 agent nodes.

By default k3d clusters come with Traefik ingress controller, we are going to disable it in favor of nginx. Last but not least, ingress controller ports `80` & `443` will be exposed.

All this settings are already defined in `kd3-lab-yaml` file, to create and start the cluster just use the configuration definition:

```bash
$ cd 00-lab
$ $env:PWD=$PWD # Only for Windows PowerShell
$ k3d cluster create --config k3d-lab.yaml
INFO[0000] Using config file k3d-lab.yaml (k3d.io/v1alpha3#simple)
INFO[0000] portmapping '80:80' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
INFO[0000] portmapping '443:443' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
INFO[0000] Prep: Network
INFO[0000] Created network 'k3d-lab'
INFO[0000] Created volume 'k3d-lab-images'
INFO[0000] Starting new tools node...
INFO[0000] Starting Node 'k3d-lab-tools'
INFO[0001] Creating node 'k3d-lab-server-0'
INFO[0001] Creating node 'k3d-lab-agent-0'
INFO[0001] Creating node 'k3d-lab-agent-1'
INFO[0001] Creating node 'k3d-lab-agent-2'
INFO[0001] Creating LoadBalancer 'k3d-lab-serverlb'
INFO[0001] Using the k3d-tools node to gather environment information
INFO[0001] HostIP: using network gateway 172.18.0.1 address
INFO[0001] Starting cluster 'lab'
INFO[0001] Starting servers...
INFO[0001] Starting Node 'k3d-lab-server-0'
INFO[0005] Starting agents...
INFO[0005] Starting Node 'k3d-lab-agent-2'
INFO[0005] Starting Node 'k3d-lab-agent-0'
INFO[0005] Starting Node 'k3d-lab-agent-1'
INFO[0015] Starting helpers...
INFO[0015] Starting Node 'k3d-lab-serverlb'
INFO[0022] Injecting '172.18.0.1 host.k3d.internal' into /etc/hosts of all nodes...
INFO[0022] Injecting records for host.k3d.internal and for 5 network members into CoreDNS configmap...
INFO[0023] Cluster 'lab' created successfully!
INFO[0023] You can now use it like this:
kubectl cluster-infonode
```

And that's all, if everything went ok one should be able to interact to the local cluster through `kubectl` command:

```bash
$ kubectl get nodes
NAME               STATUS   ROLES                  AGE   VERSION
k3d-lab-server-0   Ready    control-plane,master   55s   v1.22.6+k3s1
k3d-lab-agent-0    Ready    <none>                 48s   v1.22.6+k3s1
k3d-lab-agent-1    Ready    <none>                 48s   v1.22.6+k3s1
k3d-lab-agent-2    Ready    <none>                 48s   v1.22.6+k3s1
```

Finally check if the nginx ingress controller has its endpoint available, note that it could take some time to be available:

```bash
$ google-chrome http://localhost
Opening in existing browser session.

$ curl http://localhost
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

One should see a nginx 404 error page, that's expected and fine.

### Troubleshooting

If you face any error creating the cluster or accessing to the ingress endpoint be aware the cluster will bind ports `80` & `443` to localhost, so the ports should be available.

Last but not least, check if there is any firewall or security software that could affect it. If so, please disable it and repeat the proccess.
