---
layout: post
title: "Manually deploying Prometheus and Grafana to Proxmox LXC"
date: 2025-07-06 21:00:00 -0000
categories: runbooks
---

# Manually deploying Prometheus and Grafana to Proxmox LXC

The following deploys Prometheus and Grafana:

1. Install node exporter on test device (macbook air)
    1. running /Users/hassan/Downloads/node_exporter-1.9.1.darwin-arm64/node_exporter, listening on port 9100 and available via its IP 
    
    ```bash
    ./node_exporter --web.listen-address='192.168.8.220:9100'
    ```
    
2. Install and configure promotheus on the host (running as a container)
    1. create lxc container running debian 12
    2. access lxc shell, configure and install prometheus (installed via the .tar.gz file manually)
        
        ```bash
        apt update
        curl 192.168.8.220:9100/metrics #test connectivity
        
        useradd --no-create-home -shell /usr/sbin/nologin prometheus
        
        mkdir -p /etc/prometheus /var/lib/prometheus
        chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
        
        curl -sLO https://github.com/prometheus/prometheus/releases/download/v3.4.2/prometheus-3.4.2.linux-amd64.tar.gz
        tar -xzf prometheus-3.4.2.linux-amd64.tar.gz
        
        cd prometheus-3.4.2.linux-amd64/
        cp prometheus promtool /usr/local/bin/
        chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
        
        cp prometheus.yml /etc/prometheus/
        chown -R prometheus:prometheus /etc/prometheus/
        
        Add /etc/systemd/system/prometheus.service:
        
        [Unit]
        Description=Prometheus
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=prometheus
        ExecStart=/usr/local/bin/prometheus \
          --config.file=/etc/prometheus/prometheus.yml \
          --storage.tsdb.path=/var/lib/prometheus/
        Restart=on-failure
        
        [Install]
        WantedBy=multi-user.target
        
        systemctl daemon-reload
        systemctl enable --now prometheus
        systemctl status prometheus
        
        edit /etc/prometheus/prometheus.yml to include under targets:
        
        192.168.8.220:9100
        ```
        
3. Install and configure grafana on the host (running with a container)
    1. create lxc container running debian 12
    2. access lxc shell, configure and install grafana (installed manually/standalone binary)
        
        ```bash
        wget https://dl.grafana.com/enterprise/release/grafana-enterprise-12.0.2.linux-amd64.tar.gz
        tar -zxvf grafana-enterprise-12.0.2.linux-amd64.tar.gz
        
        useradd -r --no-create-home --shell /usr/sbin/nologin grafana #added the -r flag but didn't add it before
        
        mv /var/tmp/grafana-v12.0.2 /usr/local/grafana
        
        chown -R grafana:grafana /usr/local/grafana
        
        touch /etc/systemd/system/grafana-server.service
        
        [Unit]
        Description=Grafana Server
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=grafana
        Group=grafana
        ExecStart=/usr/local/grafana/bin/grafana-server \
          --homepath=/usr/local/grafana
        Restart=on-failure
        
        [Install]
        WantedBy=multi-user.target
        
        systemctl daemon-reload
        systemctl enable --now grafana-server
        ```
        
4. Configuring grafana to ingest from prometheus
    1. Edit in /usr/local/grafana/conf/provisioning/datasources 
    
    ```bash
    cd /usr/local/grafana/conf/provisioning/datasources
    touch grafana.yaml
    chown grafana:grafana grafana.yaml
    
    edit grafana.yaml:
    apiVersion: 1
    
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        # Access mode - proxy (server in the UI) or direct (browser in the UI).
        url: http://192.168.8.152:9090
        jsonData:
          httpMethod: POST
          manageAlerts: true
          prometheusType: Prometheus
          prometheusVersion: 2.44.0
          cacheLevel: 'High'
          disableRecordingRules: false
          incrementalQueryOverlapWindow: 10m
          exemplarTraceIdDestinations:
            # Field with internal link pointing to data source in Grafana.
            # datasourceUid value can be anything, but it should be unique across all defined data source uids.
            - datasourceUid: default_jaeger_uid
              name: traceID
    
            # Field with external link.
            - name: traceID
              url: 'http://localhost:3000/explore?orgId=1&left=%5B%22now-1h%22,%22now%22,%22Jaeger%22,%7B%22query%22:%22$${__value.raw}%22%7D%5D'
              
    curl -X POST \
      -u admin:your_password \
      http://localhost:3000/api/admin/provisioning/datasources/reload
    ```
