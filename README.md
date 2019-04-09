# Prometheus-Postgress-exporter

Using Prometheus Monitoring tool for Postgresql 

We are going to use Prometheus Monitoring tool with the OCP 3.10 to monitor the Postgresql.

Steps :-

Install Prometheus monitoring tool in your cluster.
Use the db project :- 

[root@master-0 ~]# oc project psql
Now using project "psql" on server "https://example.example.com:443".

[root@master-0 ~]# oc get pods
NAME                 READY     STATUS    RESTARTS   AGE
postgresql-1-lzqwh   1/1       Running   0          19h

[root@master-0 ~]# oc get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
postgresql   ClusterIP   172.30.133.86   <none>        5432/TCP   19h

Use the Project openshift-metrics

[root@master-0 ~]# oc project openshift-metrics 

Below the DeploymentConfig of postgres-exporter.

The following deploymentconfig will monitor the sampledb database present in postgresql-1-lzqwh pod which has  user “userTBW” and password “VXynx2vI13GX3yGM” exposed on   172.30.133.86 on port 5432 as we can see from svc of postgres above.

~~~
postgresql://USER:PASSWORD@IP:PORT/DATABASE_NAME?sslmode=disable
~~~

The above format is format of dataStore we need to monitor.


DeploymentConfig Object Definition:

apiVersion: v1
items:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    creationTimestamp: 2019-02-24T17:52:16Z
    generation: 2
    name: prometheus-postgres-exporter
    namespace: openshift-metrics
    resourceVersion: "72531"
    selfLink: /apis/apps.openshift.io/v1/namespaces/openshift-metrics/deploymentconfigs/prometheus-postgres-exporter
    uid: ec7d0dd9-385c-11e9-965d-fa163ec39299
  spec:
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      deployment-config.name: prometheus-postgres-exporter
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          deployment-config.name: prometheus-postgres-exporter
      spec:
        containers:
        - env:
          - name: DATA_SOURCE_NAME
            value: postgresql://userTBW:VXynx2vI13GX3yGM@172.30.133.86:5432/sampledb?sslmode=disable
          image: wrouesnel/postgres_exporter
          imagePullPolicy: Always
          name: default-container
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
    Triggers:


Expose Deployment to create the service.
[root@master-0 ~]# oc expose dc prometheus-postgres-exporter --port 9187

TO CHECK METRICS ON CMD:

[root@master-0 ~]# curl 172.30.75.184:9187/metrics
                                            |                     |--------->>>--|
                            ServiceIP of postgres-exporter           |
                                                                                        Port of postgres-exporter

Change the prometheus configuration so that it can listen to exporter:
Edit the configMap op prometheus. For that you need admin privileges of openshift-metrics. Add the postgressql part in configuration. I have added in last of the configuration. We can see below after changing the default configMap how it looks:

[root@master-0 ~]# oc edit cm prometheus -n openshift-metrics
apiVersion: v1
data:
  prometheus.rules: |
    groups:
    - name: example-rules
      interval: 30s # defaults to global interval
      rules:
  prometheus.yml: |
    rule_files:
      - '*.rules'
 
 
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
 
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - default
 
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kubernetes;https
 
    - job_name: 'kubernetes-controllers'
 
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - default
 
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kubernetes;https
      - source_labels: [__address__]
        action: replace
        target_label: __address__
        regex: (.+)(?::\d+)
        replacement: $1:8444
 
    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      metric_relabel_configs:
      - source_labels: [__name__]
        action: drop
        regex: 'openshift_sdn_pod_(setup|teardown)_latency(.*)'
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
 
    - job_name: 'kubernetes-cadvisor'
 
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      metrics_path: /metrics/cadvisor
 
      kubernetes_sd_configs:
      - role: node
 
      metric_relabel_configs:
      - source_labels: [__name__]
        action: drop
        regex: 'container_(cpu_user_seconds_total|cpu_cfs_periods_total|memory_usage_bytes|memory_swap|memory_working_set_bytes|memory_cache|last_seen|fs_(read_seconds_total|write_seconds_total|sector_(.*)|io_(.*)|reads_merged_total|writes_merged_total)|tasks_state|memory_failcnt|memory_failures_total|spec_memory_swap_limit_bytes|fs_(.*)_bytes_total|spec_(.*))'
 
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
 
    - job_name: 'kubernetes-service-endpoints'
 
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
 
      kubernetes_sd_configs:
      - role: endpoints
 
      relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          action: keep
          regex: 'default|metrics|kube-.+|openshift|openshift-.+'
        - source_labels: [__meta_kubernetes_namespace]
          action: drop
          regex: 'logging'
        - source_labels: [__meta_kubernetes_service_name]
          action: drop
          regex: 'prometheus-node-exporter'
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: (.+)(?::\d+);(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
 
    - job_name: 'kubernetes-logging-service-endpoints'
 
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - 'logging'
 
      relabel_configs:
        - source_labels: [__meta_kubernetes_service_name]
          action: drop
          regex: 'prometheus-node-exporter'
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: (.+)(?::\d+);(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
 
    - job_name: 'kubernetes-nodes-exporter'
 
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
 
      kubernetes_sd_configs:
      - role: node
 
      metric_relabel_configs:
      - source_labels: [__name__]
        action: drop
        regex: 'node_cpu|node_(disk|scrape_collector)_.+'
      - source_labels: [__name__]
        action: replace
        regex: '(node_(netstat_Ip_.+|vmstat_(nr|thp)_.+|filesystem_(free|size|device_error)|network_(transmit|receive)_(drop|errs)))'
        target_label: __name__
        replacement: renamed_$1
      - source_labels: [__name__]
        action: drop
        regex: 'node_(netstat|vmstat|filesystem|network)_.+'
      - source_labels: [__name__]
        action: replace
        regex: 'renamed_(.+)'
        target_label: __name__
        replacement: $1
      - source_labels: [__name__, device]
        action: drop
        regex: 'node_network_.+;veth.+'
      - source_labels: [__name__, mountpoint]
        action: drop
        regex: 'node_filesystem_(free|size|device_error);([^/].*|/.+)'
 
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
      - source_labels: [__meta_kubernetes_node_label_kubernetes_io_hostname]
        target_label: __instance__
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
 
    - job_name: 'openshift-template-service-broker'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        server_name: apiserver.openshift-template-service-broker.svc
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - openshift-template-service-broker
 
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: apiserver;https
 
    - job_name: 'openshift-router'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        server_name: router.default.svc
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
 
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - default
 
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: router;1936-tcp
 
    - job_name: 'postgreSQL'
      scheme: http
      static_configs:
      - targets:
        - "172.30.75.184:9187"
 
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "localhost:9093"
kind: ConfigMap
metadata:
  creationTimestamp: 2019-02-24T14:01:33Z
  name: prometheus
  namespace: openshift-metrics
  resourceVersion: "93255"
  selfLink: /api/v1/namespaces/openshift-metrics/configmaps/prometheus
  uid: b16df417-383c-11e9-965d-fa163ec39299



Here are the some screenshots of metrics in prometheus below: 






