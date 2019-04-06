+++
tags = ["Prometheus", "Grafana", "Kubernetes"]
series = []
series_part = 0
title = "Dynamic Configuration Discovery in Grafana"
publishDate = "2019-03-25"

+++

A few of my colleagues have [written](https://blog.scottlowe.org/2019/02/01/scraping-envoy-with-prometheus-operator/) [posts](https://jpweber.io/blog/monitor-external-services-with-the-prometheus-operator/) recently on the Prometheus stack so I thought I'd get in on the action.

In this post I'll walk through how [Grafana](https://grafana.com/) uses sidecar containers to dynamically discover datasources and dashboards declared as ConfigMaps in Kubernetes to allow easy and extensible configuration for cluster operators.

Let's dive in!

<!--more-->

For the following examples I used the Prometheus Operator [Helm chart](https://github.com/helm/charts/blob/master/stable/prometheus-operator) which installs the operator stack (Prometheus, Grafana, Alertmanager, CRDs, RBAC resources, etc...) including some default dashboards and alerts as defined by the awesome [kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus) project.

## Datasources 

Grafana is a tool for visualizing data, and can be connected to multiple query sources. The way that Grafana connects (URL, credentials, etc...) to backend datasources is configured using one or more configuration files which Grafana reads during provisioning. The chart / operator makes this even easier by giving us a neat way of defining Kubernetes ConfigMaps with our datasource configuration and using a sidecar to drop them into the correct directory (`provisioning/datasources`) for Grafana to use.

If you're following along at home, make sure you've enabled these features in the chart `values.yaml`:

```yaml
grafana:
  sidecar:
    datasources:
      enabled: true
      label: grafana_datasource
```

The label key `grafana_datasource` is default, but if you change this make sure to remember as we'll need this value later when creating our ConfigMap.

Now let's take a look at the Grafana deployment itself:

```yaml
# Source: prometheus-operator/charts/grafana/templates/deployment.yaml
# ...
- name: grafana
  image: "grafana/grafana:6.0.0"
  imagePullPolicy: IfNotPresent
  volumeMounts:
    - name: config
      mountPath: "/etc/grafana/grafana.ini"
      subPath: grafana.ini
    - name: sc-datasources-volume
      mountPath: "/etc/grafana/provisioning/datasources"
# ...
```

We're mounting the `sc-datasources-volume` to the `provisioning-datasources` directory in the Grafana container. That volume is being declared further down the deployment as an `emptyDir`:

```yaml
volumes:
  - name: sc-datasources-volume
    emptyDir: {}
```

Now the neat part is an `initContainer` ([learn more](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)) which is defined as part of the Grafana pod:

```yaml
initContainers:
  - name: grafana-sc-datasources
    image: "kiwigrid/k8s-sidecar:0.0.11"
    imagePullPolicy: IfNotPresent
    env:
      - name: METHOD
        value: LIST
      - name: LABEL
        value: "grafana_datasource"
      - name: FOLDER
        value: "/etc/grafana/provisioning/datasources"
    volumeMounts:
      - name: sc-datasources-volume
        mountPath: "/etc/grafana/provisioning/datasources"
```

This is running an image called `kiwigrid/k8s-sidecar` and setting some `env` variables. Let's take a look at what the [sidecar does](https://github.com/kiwigrid/k8s-sidecar/blob/master/sidecar/sidecar.py). It's actually a pretty cool (but simple, less than 200 lines of python!) utility that talks to the Kubernetes API and watches a configurable set of namespaces for ConfigMaps with a specific label (this is the `grafana_datasource` label we set earlier in the `values.yaml`). It then grabs the data in the ConfigMaps and writes to a corresponding file in the `FOLDER` that we pass in.

Now for the final piece, let's take a look at the default ConfigMap generated as part of the operator / chart:

```yaml
# Source: prometheus-operator/templates/grafana/configmaps-datasources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-operator-grafana-datasource
  labels:
    grafana_datasource: "1"
    app: prometheus-operator-grafana
    chart: prometheus-operator-4.3.1
    release: "prometheus-operator"
    heritage: "Tiller"
data:
  datasource.yaml: |-
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-operator-prometheus:9090/
      access: proxy
      isDefault: true
```

We can see our `grafana_dashboard` label, and the contents of the ConfigMap specify that Grafana should query Prometheus on it's internal Kubernetes service URL / port. Lastly if we look at the logs for the `initContainer` we can see it working and discovering the ConfigMap:

```txt
Starting config map collector
Config for cluster api loaded...
Working on configmap monitoring/prometheus-operator-etcd
Working on configmap monitoring/prometheus-operator-grafana
Working on configmap monitoring/prometheus-operator-grafana-config-dashboards
Working on configmap monitoring/prometheus-operator-grafana-datasource
Configmap with label found
Working on configmap monitoring/prometheus-operator-k8s-cluster-rsrc-use
Working on configmap monitoring/prometheus-operator-k8s-coredns
```

_**Note:** Grafana only checks it's `provisioning/datasources` directory on startup, which is why the sidecar is implemented as an `initContainer` in this case, rather than a long-running watcher. This can lead to race conditions where the datasource ConfigMap(s) may not have been applied before the `initContainer` runs, leaving the datasources section in Grafana empty._

{{% top %}}

## Dashboards

Grafana is super flexible and there is almost no limit to the variety of dashboards that you can create. Dashboards in Grafana are [defined as JSON](http://docs.grafana.org/reference/dashboard/) and Grafana looks for them by reading a provider configuration file:

```yaml
# Source: prometheus-operator/charts/grafana/templates/configmap-dashboard-provider.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: grafana
    chart: grafana-2.2.0
    release: prometheus-operator
    heritage: Tiller
  name: prometheus-operator-grafana-config-dashboards
data:
  provider.yaml: |-
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      options:
        path: /tmp/dashboards
```

We can see that `/tmp/dashboards` is where we need to put our dashboard definitions for Grafana to pick them up, and Grafana again uses the sidecar model to allow us to define dashboards as Kubernetes ConfigMap objects. Make sure that dynamic dashboards is enabled in the chart's `values.yaml`:

```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
```

Now let's look at the relevant section of the Grafana deployment:

```yaml
containers:
  - name: grafana-sc-dashboard
    image: "kiwigrid/k8s-sidecar:0.0.11"
    imagePullPolicy: IfNotPresent
    env:
      - name: LABEL
        value: "grafana_dashboard"
      - name: FOLDER
        value: "/tmp/dashboards"
    volumeMounts:
      - name: sc-dashboard-volume
        mountPath: "/tmp/dashboards"
```

There are a few changes from the sidecar configuration used for datasources. Firstly, we are not using an `initContainer`, but defining another container that will run indefinitely alongside the actual Grafana application container. The reason for this is that unlike datasources, Grafana will check for new dashboards regularly, so our sidecar is constantly polling for new ConfigMaps and writing them to the `/tmp/dashboards` directory. In order to make the sidecar behave more like a daemon in this regard (rather than a one-shot process as before), we omit the `METHOD` environment variable from it's definition. A quick look at the code shows how this effects the behaviour:

```python
# ...
k8s_method = os.getenv("METHOD")
if k8s_method == "LIST":
    listConfigmaps(label, targetFolder, url, method, payload, namespace, folderAnnotation)
else:
    while True:
        try:
            watchForChanges(label, targetFolder, url, method, payload, namespace, folderAnnotation)
# ...
```

Secondly we are changing the `LABEL` that should be watched (`grafana_dashboard` instead of `grafana_datasource`). Finally we are specifying the dashboard directory (`/tmp/dashboards`) rather than the `provisioning/datasources` directory.

The chart / operator includes a number of pre-built dashboards that can be found in the `templates/grafana/dashboards` directory. Here's a snipper from the `nodes.yaml` dashboard:

```yaml
# Source: prometheus-operator/templates/grafana/dashboards/nodes.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-operator-nodes
  labels:
    grafana_dashboard: "1"
    app: prometheus-operator-grafana
    chart: prometheus-operator-4.3.1
    release: "prometheus-operator"
    heritage: "Tiller"
data:
  nodes.json: |-
    {
        "__inputs": [

        ],
        "__requires": [

        ],
        "annotations": {
            "list": [

            ]
        },
        "editable": false,
# ...
```

A quick look at the logs for the `grafana-sc-dashboard` container shows the sidecar doing it's work:

```txt
Starting config map collector
Config for cluster api loaded...
Working on configmap monitoring/prometheus-operator-statefulset
Configmap with label found
File in configmap statefulset.json ADDED
Working on configmap monitoring/prometheus-operator-grafana-datasource
Working on configmap monitoring/prometheus-operator-k8s-resources-namespace
Configmap with label found
File in configmap k8s-resources-namespace.json ADDED
Working on configmap monitoring/prometheus-operator-grafana
Working on configmap monitoring/prometheus-operator-grafana-config-dashboards
Working on configmap monitoring/prometheus-operator-k8s-coredns
Configmap with label found
File in configmap k8s-coredns.json ADDED
Working on configmap monitoring/prometheus-operator-k8s-resources-cluster
Configmap with label found
File in configmap k8s-resources-cluster.json ADDED
```

I hope this has been a useful introduction to how dynamic configuration discovery works in Grafana for datasources and dashboards. Feel free to share using the button below and [contact me on Twitter](https://twitter.com/johnharris85) if you have questions or comments on this post or suggestions for future posts!