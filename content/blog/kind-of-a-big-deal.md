+++
title = "Kubernetes in Docker: Kind of a Big Deal"
tags = ["Kind", "Kubernetes", "Kubeadm", "Bootstrapping", "Docker"]
series = []
series_part = 0
draft = true
publishDate = "2019-04-05"

+++

I've been playing a little bit with the [Cluster API](https://github.com/kubernetes-sigs/cluster-api) project recently (posts on that coming soon), and using [Kind](https://github.com/kubernetes-sigs/kind) as an ephemeral bootstrap cluster. Kind is a super cool and fairly new project that I figured I'd explore a little bit in this post as some folks may not be aware of it or had a chance to get hands-on with it.

Kind was born out of the neccessity for a lightweight local Kubernetes setup that could be used for testing and conformance. It has uses now across several SIGs and the [goals of the project](https://kind.sigs.k8s.io/docs/design/principles/) are laid out in the official docs.

<!--more-->

## What is Kind?

Kind is a tool that allows you to spin up Kubernetes clusters locally, using containers as 'nodes'. The images it uses are full base images containing everything required to run Kubernetes control plane and worker nodes. Systemd, Docker, kubelet, the works! This _does_ mean the images can be a little heavy (~1.5GB) but it's still a great way to run Kubernetes locally in a multi-node configuration without having the overhead of running multiple virtual machines.

## Getting started

To get started all you need is to install [Docker](https://docs.docker.com/install/) (>= 18.09.1) and download the [kind](https://github.com/kubernetes-sigs/kind/releases) binary (I'm using 0.2.1).

Let's begin by starting a minimal cluster:

```txt
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.13.4) 🖼
 ✓ Preparing nodes 📦
 ✓ Creating kubeadm config 📜
 ✓ Starting control-plane 🕹️
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl cluster-info
```

Wow that was pretty fast! Let's do some sanity testing on this cluster and deploy a sample app:

```txt
$ export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"

$ kubectl create deployment test-deploy --image johnharris85/simple-hostname-reporter:3
deployment.apps/test-deploy created

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
test-deploy   1/1     1            1           49s

$ kubectl port-forward $(kubectl get pods -l "app=test-deploy" -oname) 5000:5000 &
Forwarding from 127.0.0.1:5000 -> 5000

$ curl localhost:5000
Handling connection for 5000
<h1>Path / served from host : test-deploy-79cdfbb8d9-24w6n</h1>
```

OK so far, so good. We have deployed a simple cluster and deployed a test application to it. By default kind gives us a one-node (the control plane) cluster called 'kind'. Now let's configure a slightly larger cluster and then see what additional options we have.

Kind defines a configuration file format and can pass a config file in at runtime. We're going to take the [sample](https://raw.githubusercontent.com/kubernetes-sigs/kind/master/site/content/docs/user/kind-example-config.yaml) config and alter it a little. Below is the config we're going to use:

```yaml
# kind uses a k8s-like configuration format
apiVersion: kind.sigs.k8s.io/v1alpha2
kind: Config

# define our nodes, here we're using 3 control-plane nodes for an 'HA' setup and a single worker
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
```

First let's delete our first cluster:

```txt
$ kind delete cluster
```

Now save the file above as `kind-config` and run:

```txt
$ kind create cluster --config kind-config --name test-kind-cluster --image kindest/node:v1.14.0
```

Here we're explicitly giving the cluster a name as well as providing a specific image (I want to stand up a 1.14 cluster), and specifying our config file. Once that's all finished provisioning, let's go spelunking around with Docker to see inside the nodes (outputs truncated here for formatting).

```txt
$ docker ps
CONTAINER ID        IMAGE                         PORTS                                  NAMES
98862b25d92a        kindest/node:v1.14.0          39003/tcp, 127.0.0.1:39003->6443/tcp   test-kind-cluster-control-plane
1b0e54acdb1b        kindest/node:v1.14.0          33997/tcp, 127.0.0.1:33997->6443/tcp   test-kind-cluster-control-plane3
6b5e02d9efd6        kindest/node:v1.14.0          45791/tcp, 127.0.0.1:45791->6443/tcp   test-kind-cluster-control-plane2
62bb336ebd77        kindest/node:v1.14.0                                                 test-kind-cluster-worker
005632f4edc5        kindest/node:v1.14.0          35143/tcp, 0.0.0.0:35143->6443/tcp     test-kind-cluster-external-load-balancer
```

We can see that each 'node' in Kubernetes is a Docker container, and in this case Kind has spun up a load balancer container as we specified multiple control-plane nodes. We can take a look at the HAProxy configuration inside the container: and see that it's pointing to the internal bridge IPs for each of our control-plane Docker containers:

```txt
$ docker exec -it test-kind-cluster-external-load-balancer cat /kind/haproxy.cfg
...
frontend controlPlane
    bind *:6443
    option tcplog
    mode tcp
    default_backend kube-apiservers

backend kube-apiservers
    mode tcp
    balance roundrobin
    option ssl-hello-chk

    server test-kind-cluster-control-plane 172.17.0.3:6443 check
    server test-kind-cluster-control-plane2 172.17.0.4:6443 check
    server test-kind-cluster-control-plane3 172.17.0.2:6443 check
...
```

We can then verify those IPs are actually the IPs for each control-plane Docker container on the internal bridge network:

```txt
$ docker ps -q --filter "name=control-plane" | xargs docker inspect --format '{{ .NetworkSettings.IPAddress }} {{ .Name }}'
172.17.0.4 /test-kind-cluster-control-plane2
172.17.0.3 /test-kind-cluster-control-plane
172.17.0.2 /test-kind-cluster-control-plane3
```

_**Note:** While digging through the kubeconfig in this setup, I noticed that it didn't actually point to the load balancer, but instead to the first control-plane node which was kind of confusing! Turns out that it was a bug that's [now been fixed](https://github.com/kubernetes-sigs/kind/pull/418/files)._

We know that Kind is using Kubeadm to bootstrap the cluster, so let's take a look at the Kubeadm configuration that's being used:

```txt
$ docker exec -it test-kind-cluster-control-plane cat /kind/kubeadm.conf
```

```yaml
...
apiServer:
  certSANs:
  - localhost
apiVersion: kubeadm.k8s.io/v1beta1
clusterName: test-kind-cluster
controlPlaneEndpoint: 172.17.0.2:6443
controllerManager:
  extraArgs:
    enable-hostpath-provisioner: "true"
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
metadata:
  name: config
...
```

Now let's say we want to modify the Kubeadm configuration in order to add an admission controller to the APIServer. Kind allows us to specify that through through the Kind configuration file:

```yaml
apiVersion: kind.sigs.k8s.io/v1alpha3
kind: Cluster

nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker

kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta1
  kind: ClusterConfiguration
  metadata:
    name: config
  apiServer:
    extraArgs:
      enable-admission-plugins: "AlwaysPullImages,NodeRestriction"
```

Kind uses Kustomize to allow alterations to the Kubeadm configuration with the `kubeadmConfigPatches` key equivalent to `patchesStrategicMerge` in Kustomize. There is also a key `kubeadmConfigPatchesJson6902` in the Kind configuration which is equivalent to Kustomize's `patchesJson6902`.

{{% top %}}

## Digging Deeper & Base Images

Now we have a good idea of what Kind does and how to configure some more complex clusters, let's take a look a bit deeper and see some of things it's doing under the hood. One really useful property of Kind is that once you have the node image (`kindest/node:xxx`) on your system, it can run completely offline. It does this by pre-pulling required images (like `etcd` and `weave`) during the node build process and storing them inside the node at `/kind/images` so they're available later during bootstrapping. 

If we take a look at [`pkg/build/node/const.go`](https://github.com/kubernetes-sigs/kind/blob/master/pkg/build/node/const.go) we can see the default CNI images (and manifest) that will be applied during bootstrapping:

```go
var defaultCNIImages = []string{"weaveworks/weave-kube:2.5.1", "weaveworks/weave-npc:2.5.1"}
```

Then in [`pkg/build/node/node.go`](https://github.com/kubernetes-sigs/kind/blob/master/pkg/build/node/node.go) we can see those images being added to the slice of other images required by Kubeadm, then pulled and stored in the `images` directory:

```go
requiredImages = append(requiredImages, defaultCNIImages...)
...
pulled := []string{}
    for i, image := range requiredImages {
        if !builtImages.Has(image) {
            fmt.Printf("Pulling: %s\n", image)
            err := docker.Pull(image, 2)
            if err != nil {
                return err
            }
            // TODO(bentheelder): generate a friendlier name
            pullName := fmt.Sprintf("%d.tar", i)
            pullTo := path.Join(imagesDir, pullName)
            err = docker.Save(image, pullTo)
            if err != nil {
                return err
            }
            pulled = append(pulled, fmt.Sprintf("/build/bits/images/%s", pullName))
        }
    }
```

These then end up as a `tar` in `/kind/images` and are loaded during node creation in [`pkg/cluster/nodes/node.go`](https://github.com/kubernetes-sigs/kind/blob/master/pkg/clusters/nodes/node.go):

```go
// LoadImages loads image tarballs stored on the node into docker on the node
func (n *Node) LoadImages() {
    // load images cached on the node into docker
    if err := n.Command(
        "/bin/bash", "-c",
        // use xargs to load images in parallel
        `find /kind/images -name *.tar -print0 | xargs -0 -n 1 -P $(nproc) docker load -i`,
    ).Run(); err != nil {
        log.Warningf("Failed to preload docker images: %v", err)
        return
    }
...
```

We can also check out the directory in the container and see that the compressed images match the ones specified in the default CNI declaration above:

```txt
$ docker exec -it test-kind-cluster-control-plane tar xf /kind/images/7.tar repositories -O
{"weaveworks/weave-kube":{"2.5.1":"c6d9032a7318819e4b415a2c7986754779875eb2866e384e9cf5f058925b192d"}}
```

### Clean-up

To clean-up you just need to delete the Kind cluster (you can also optionally delete the `kindest` images if you want):

```txt
$ kind delete cluster --name test-kind-cluster

$ docker image rm $(docker image ls --filter=reference='kindest/*' --format '{{ .ID }}')
```

\
If you'd like to learn more about Kind and are in the San Francisco area on Monday 8th April 2019, you should definitely check out my colleague [@mauilion](https://twitter.com/mauilion)'s talk at the [Kubernetes meetup](https://www.anaplan.com/events/opencensus-kind-and-embracing-cloud-native/) for more detail and some cool tips & tricks!

I hope this has been a useful introduction to Kind and some of the configuration that is possible. Feel free to share using the button below and [contact me on Twitter](https://twitter.com/johnharris85) if you have questions or comments on this post or suggestions for future posts!