+++
title = "Least Privilege in Kubernetes Using Impersonation"
tags = ["Impersonation", "Kubernetes", "RBAC", "Authorization", "Authentication", "Kubectl"]
series = []
series_part = 0
publishDate = "2019-08-04"
headerImage = "fregoli.jpg"

+++

Recently I implemented an auth[zn] solution for a customer using [Dex](https://github.com/dexidp/dex) & AD. I might write more about that implementation in another post (as there were some interesting new capabilities we needed to add to Dex for our use case), but in this post I'm going to cover the pretty simple but powerful RBAC setup that we designed and implemented to compliment it.

Kubernetes supports the concept of ['impersonation'](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation) and we're going to look at the user & group configuration that we created using impersonation to enable a least-privilege type of access to the cluster, even as an administrator, to ensure that it was more difficult to accidentally perform unwanted actions, while keeping the complexity level low.

<!--more-->

## What are we trying to do?

In our example Kubernetes cluster we have a multi-tenant environment with each application development team having their own namespace. Our developers aren't _super_ familiar with Kubernetes, so while we want to give them admin access to their namespace, most of their use cases really just involve viewing the state, rather than changing things. In this example we only have one team, the 'app-team'.

We also have an 'ops-team', who are administrators over the whole cluster. Even though they have all this power, they too also mainly just look at the state of the cluster rather than editing things. It would be great if we could design a solution that allows everyone _view only_ rights on the cluster by default, but allows them to 'elevate' (kind of like [`sudo`](https://en.wikipedia.org/wiki/Sudo) for Unix-like systems) when they need to make changes.

Enter impersonation!

![](/images/fregoli.jpg)

Using impersonation, we're going to setup a system where there will be `admin` roles for each application's namespace, however we will _not_ be giving this role directly to the `app-team` group. Instead we will be adding creating an `app-team-admin` user that has the role bound to them. Then we can give permissions for our `app-team` group to _impersonate_ that `app-team-admin` user. We also bind a `view-only` role to the `app-team` group so that by default, all their regular `kubectl` commands will be enacted with read-only permissions. If they should need to change things in the cluster, they can impersonate the admin by using `--as=app-team-admin` with their `kubectl` commands.

We will setup a similar system for the `ops-team`, making use of the `cluster-admin` ClusterRole that already exists and binding it to a `cluster-admin` user that we will allow members of the `ops-team` group to impersonate / assume.

{{% top %}}

## Configuring the Ops-Team

Creating the user credentials is out of scope for this post, and I already have a kubeconfig (`alice-kfg`) created that will let me access the cluster with the user `alice`, and that contains her group information (`ops-team`).

First we're going to create the ClusterRoleBinding that allows users in the `ops-team` group access to the `view` built-in ClusterRole, this is what gives us our default read-only access:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: ops-team
```

Now we create a ClusterRoleBinding to allow our `cluster-admin` user to have `cluster-admin` ClusterRole permissions on the cluster. Remember we're not binding this ClusterRole directly to our `ops-team` group:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: cluster-admin
```

Finally we create a ClusterRole called `cluster-admin-impersonator` that allows the impersonation of the `cluster-admin` user, and a ClusterRoleBinding that binds that capability to everyone in the `ops-team` group:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-impersonator
rules:
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["cluster-admin"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-impersonate
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin-impersonator
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: ops-team
```

Now let's apply all the RBAC resources and test with our `alice-kfg`:

```txt
$ kubectl apply -f ops-team/
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-view created
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-crb created
clusterrole.rbac.authorization.k8s.io/cluster-admin-impersonator created
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-impersonate created

$ KUBECONFIG=alice-kfg kubectl get configmaps
No resources found.

$ KUBECONFIG=alice-kfg kubectl create configmap my-config --from-literal=test=test
Error from server (Forbidden): configmaps is forbidden: User "alice" cannot create resource "configmaps" in API group "" in the namespace "default"
```

OK so reading works (but we don't have any configmaps in our `default` namespace), but we can't create anything using Alice yet. Now let's impersonate our `cluster-admin` user (note the `--as=cluster-admin` argument to kubectl) when we want to change the cluster:

```txt
KUBECONFIG=alice-kfg kubectl --as=cluster-admin create configmap my-config --from-literal=test=test
configmap/my-config created
```

**Success!** 

And one of the great things about the impersonation approach is that all of this is played out in the Kubernetes audit logs, so I can see the original user log in, impersonate the cluster-admin, then take action.

{{% top %}}

## Configuring the App-Team

The `app-team` is configured in a very similiar way to our `ops-team`, but with permissions restricted to their own namespace, rather than being cluster-wide:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-team-admin
  namespace: app-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: app-team-admin
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-team-view
  namespace: app-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: app-team
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-team-impersonator
rules:
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["app-team-admin"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-team-admin-impersonate
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: app-team-impersonator
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: app-team
```

I already have another kubeconfig setup for `bob` in the `app-team` called `bob-kfg`:

```txt
$ kubectl apply -f ops-team/
rolebinding.rbac.authorization.k8s.io/app-team-admin created
rolebinding.rbac.authorization.k8s.io/app-team-view created
clusterrole.rbac.authorization.k8s.io/app-team-impersonator created
clusterrolebinding.rbac.authorization.k8s.io/app-team-admin-impersonate created

$ KUBECONFIG=bob-kfg kubectl get configmaps
Error from server (Forbidden): configmaps is forbidden: User "bob" cannot list resource "configmaps" in API group "" in the namespace "default"

$ KUBECONFIG=bob-kfg kubectl get configmaps -n app-team
No resources found.

$ KUBECONFIG=bob-kfg kubectl create configmap my-config --from-literal=test=test -n app-team --as=app-team-admin
configmap/my-config created
```

I hope this has been a useful tutorial for setting up an RBAC system for minimal privilege over a cluster, while retaining a simple-to-use workflow and auditing capabilities. Feel free to share using the button below and [contact me on Twitter](https://twitter.com/johnharris85) if you have questions or comments on this post or suggestions for future posts!
