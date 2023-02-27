---
title: "Veeam Exporter for Prometheus"
date: 2023-02-27T09:54:11Z
tags: ['Veeam', 'Backup', 'Prometheus', 'Exporter', 'Grafana']
draft: true
---

![Veeam Exporter dashboard in Grafana](https://github.com/peekjef72/veeam_exporter/raw/master/screenshots/veeam_general_dash.png "Veeam Exporter dashboard in Grafana")

Over the past couple of months I've put a considerable amount of time into deployment of a monitoring infrastructure in my home-lab that would replace Splunk and SCOM. In a way, this setup introduced a new level of monitoring which I did not have before, I've deeply fallen for metrics and the power of Prometheus and pretty much sunk into the Grafana's LGTM ecosystem, quickly implementing Tempo and Loki for a full experience. 

As I've been using Veeam for a long time and I wanted to monitor it in terms of metrics; just to get a simple overview without having to open the console. Previously, I fed the data into Splunk; however, I've decided to decommision it so a new solution was needed.

Initially, all research pointed to various InfluxDB based solutions; however, I wanted to stick to Prometheus and utilize an exporter instead. Eventually, I came across [Veeam Exporter](https://github.com/peekjef72/veeam_exporter) which was exactly what I was looking for.

Unfortunately, this exported had a dependency on Veeam Enterprise Manager; which at the time I did not have access to. With the help of friendly guys at Veeam, I've recently obtained an [NFR license](https://go.veeam.com/free-nfr-veeam-availability-suite) sized to my requirements so I was able to deploy Veeam Backup & Replication along with the Enterprise Manager.

With the dependencies in place, I noticed there is no container image. For me, this is a must as I wanted to run the exporter in my Kubernetes cluster; therefore, I went ahead and built one myself and published it to Docker Hub for others to consume until an official image is made available. The image is available at [mateuszd/veeam-exporter](https://hub.docker.com/repository/docker/mateuszd/veeam-exporter). A PR contributing the `dockerfile` back to the project is also submitted at [here](https://github.com/peekjef72/veeam_exporter/pull/10).

With this, a quick deployment was created for my cluster as shown below and voila, I've got a nice dashboard with Veeam metrics. 


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: veeam-exporter
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: veeam-exporter
  template:
    metadata:
      labels:
        app.kubernetes.io/name: veeam-exporter
    spec:
      containers:
        - name: veeam-exporter
          image: mateuszd/veeam-exporter:latest
          resources:
            limits:
              memory: "64Mi"
              cpu: "100m"
          ports:
            - containerPort: 9247
              name: http
          volumeMounts:
            - name: secret
              mountPath: /app/veeam_exporter/conf/config.yml
              subPath: config.yaml
      volumes:
        - name: secret
          secret:
            secretName: veeam-exporter-secret
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: veeam-exporter-podmonitor
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: veeam-exporter
  podMetricsEndpoints:
    - port: http
      scrapeTimeout: 60s
      relabelings:
        - action: replace
          targetLabel: __param_target
          replacement: my-backup-server
          # This refers to the VB&R server name
        - action: replace
          targetLabel: instance
          replacement: my-backup-server
          # This refers to the VEM server name
---
kind: Secret
metadata:
  name: veeam-exporter-secret
type: Opaque
apiVersion: v1
stringData:
  config.yaml: |
    veeams:
      - host: my-backup-server
        port: 9398
        user: 'my-user'
        password: 'my-password'
        protocol: https
        verify_ssl: false
    #   timeout: 20
    #   keep_session: true # default
    #   default_labels:
    #     - name: veeam_em
    #       value: my_veeam_em_server.domain
    #       proxy:
    #         url: http://my.proxy.domain:port/
    #         protocol: https
    weblisten:
      address: 0.0.0.0
      port: 9247
    logger:
      level: info
      facility: stdout
    metrics_file: "conf/metrics/*_metrics.yml"
```

Some notes about the above setup:
- Prometheus operator needs to be installed in the cluster.
- Credentials in the secret need to be granted a role in Veeam Enterprise Manager.