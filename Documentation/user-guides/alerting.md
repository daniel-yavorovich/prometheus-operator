# Alerting

This guide assumes you have a basic understanding of the `Prometheus` resource and have read the [getting started](getting-started.md).

Besides the `Prometheus` and `ServiceMonitor` resource the Prometheus Operator also introduces the `Alertmanager`. It allows declaratively describing an Alertmanager cluster. Before diving into deploying an Alertmanager cluster, it is important to understand the contract between Prometheus and Alertmanager.

The Alertmanager's features include:

* Deduplicating alerts fired by Prometheus
* Silencing alerts
* Route and send grouped notifications via providers (PagerDuty, OpsGenie, ...)

Prometheus' configuration includes so called rule files, which contain the [alerting rules](https://prometheus.io/docs/alerting/rules/). When an alerting rule triggers it fires that alert against *all* Alertmanager instances, on *every* rule evaluation interval. The Alertmanager instances communicate to each other which notifications have already been sent out. You can read more about why these systems have been designed this way in the [High Availability scheme description](../high-availability.md).

Let's create an example Alertmanager cluster, with three instances.

[embedmd]:# (../../example/user-guides/alerting/alertmanager-example.yaml)
```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: Alertmanager
metadata:
  name: example
spec:
  replicas: 3
```

The Alertmanager instances will not be able to start up, unless a valid configuration is given. This is an example configuration, that does not actually do anything as it sends notifications against a non existent `webhook`, but will allow the Alertmanager to start up. Read more about how to configure the Alertmanager on the [upstream documentation](https://prometheus.io/docs/alerting/configuration/).

[embedmd]:# (../../example/user-guides/alerting/alertmanager-example-config.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-example
data:
  alertmanager.yaml: |-
    global:
      resolve_timeout: 5m
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'webhook'
    receivers:
    - name: 'webhook'
      webhook_configs:
      - url: 'http://alertmanagerwh:30500/'
```

To be able to view the web UI, expose it via a `Service`. A simple way to do this is to use a `Service` of type `NodePort`.

[embedmd]:# (../../example/user-guides/alerting/alertmanager-example-service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-example
spec:
  type: NodePort
  ports:
  - name: web
    nodePort: 30903
    port: 9093
    protocol: TCP
    targetPort: web
  selector:
    alertmanager: example
```

Once created it allows the web UI to be accessible via a node's IP and the port `30903`.

Now this is a fully functional highly available Alertmanager cluster, but it does not get any alerts fired against it. Let's setup Prometheus instances that will actually fire alerts against it.

[embedmd]:# (../../example/user-guides/alerting/prometheus-example.yaml)
```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: Prometheus
metadata:
  name: example
spec:
  replicas: 2
  alerting:
    alertmanagers:
    - namespace: default
      name: alertmanager-example
      port: web
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
```

Prometheus rule files are held in a `ConfigMap` called `prometheus-<prometheus-object-name>-rules`. All top level files that end with the `.rules` extension will be loaded by a Prometheus instance.

[embedmd]:# (../../example/user-guides/alerting/prometheus-example-rules.yaml)
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: prometheus-example-rules
data:
  example.rules: |
    ALERT ExampleAlert
    IF vector(1)
```

> Note the Prometheus Operator will create an empty `ConfigMap` if it does not already exist.

That example `ConfigMap` always immediately triggers an alert, which is only for demonstration purposes. To validate that everything is working properly have a look at each of the Prometheus web UIs.

To be able to view the web UI without a `Service`, `kubectl`'s proxy functionality can be used.

Run:

```bash
kubectl proxy --port=8001
```

Then the web UI of each Prometheus instance can be viewed, they both have a firing alert called `ExampleAlert`, as defined in the loaded alerting rules.

* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-0:9090/alerts
* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-1:9090/alerts

Looking at the status page for "Runtime & Build Information" on the Prometheus web UI shows the discovered and active Alertmanagers that the Prometheus instance will fire alerts against.

* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-0:9090/status
* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-1:9090/status

These show three discovered Alertmanagers.

Heading to the Alertmanager web UI now shows one active alert, although all Prometheus instances are firing it. [Configuring the Alertmanager](https://prometheus.io/docs/alerting/configuration/) further allows custom alert routing, grouping and notification mechanisms.
