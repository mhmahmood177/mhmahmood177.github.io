---
layout: post
title: "Setting up Prometheus Proxmox VE Exporter to gain metrics from proxmox ✅"
date: 2025-07-09 20:55:00 -0000
categories: runbooks
---

1. Remote onto proxmox node and run the following:
    1. create proxmox user and api token 
    
    ```bash
    Datacentre > permissions > users > add (Create a user here for prometheus))
    
    Datacentre > permissions > API tokens > add (Create an API token and associate with prometheus)
    
    Datacentre > permissions > add (ensure token and user has read only access to /) 
    ```
    
     b. Installing PVE exporter in a venv 
    
    ```bash
    python3 -m venv /opt/prometheus-pve-exporter
    
    /opt/prometheus-pve-exporter/bin/pip install prometheus-pve-exporter
    
    /opt/prometheus-pve-exporter/bin/pve_exporter --help
    
    useradd -r --no-create-home --shell /usr/sbin/nologin prometheus
    
    mkdir -p /etc/prometheus
    cat <<EOF > /etc/prometheus/pve.yml
    default:
      user: "prometheus@pve"
      token_name: "promtoken"
      token_value: "changeme"
      verify_ssl: false
    EOF
    
    chown prometheus:prometheus /etc/prometheus/pve.yml
    chmod 600 /etc/prometheus/pve.yml
    
    cat <<EOF> /etc/systemd/system/prometheus-pve-exporter.service
    [Unit]
    Description=Prometheus exporter for Proxmox VE
    Documentation=https://github.com/znerol/prometheus-pve-exporter
    
    [Service]
    Restart=always
    User=prometheus
    Group=prometheus
    ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter --config.file /etc/prometheus/pve.yml
    
    ProtectSystem=full
    ProtectHome=true
    NoNewPrivileges=true
    
    [Install]
    WantedBy=multi-user.target
    EOF
    
    systemctl daemon-reload
    systemctl enable --now prometheus-pve-exporter
    
    curl http://localhost:9221/metrics
    ```
    
    c. add config to prometheus to ingest from pve node exporter
    

```bash
edit /etc/prometheus/prometheus.yml to include under targets:

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ['localhost:9090']
        labels:
          app: "prometheus"

  - job_name: "macbook-air"
    static_configs:
      - targets: ['192.168.8.220:9100']
        labels:
          app: "node-exporter"
          host: "macbook-air"

  - job_name: "proxmox"
	  metrics_path: /pve
	  params:
		  target: ['192.168.8.151']
		  cluster: ['1']
		  node: ['1'}
    static_configs:
      - targets: ['192.168.8.151:9221']
        labels:
          app: "pve-exporter"
          host: "proxmox-node"
          
curl -X POST http://localhost:9090/-/reload (restart prometheus if web enable lifecycle enabled)

or

killall -HUP prometheus

curl http://localhost:9090/api/v1/targets | jq

check you can obtain metrics in prometheus from http://192.168.8.151:9221/pve
```
