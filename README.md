![intro-image](https://raw.githubusercontent.com/ghatechteam/system-monitor-toolkit/main/image.jpeg?token=GHSAT0AAAAAAB2YD4UDZECHE7HMCLX6U2Q2Y3FGFDQ)

> Prometheus reliable solution for monitoring, scraping metrics over http from instrumented jobs and alerting of highly dynamic service-oriented architectures

> 

## Prerequisites

- Node Exporter to scrape hardware- and kernel-related metrics
- Ports Required- **9090** (Prometheus), **3000** (Grafana), **9100** (Node Exporter)

## Install Prometheus LTS version only

```bash
# creating Prometheus System Users and Directory
$ useradd --no-create-home --shell /bin/false prometheus && useradd --no-create-home --shell /bin/false node_exporter
$ mkdir /etc/prometheus
$ mkdir /var/lib/prometheus
$ chown prometheus:prometheus /etc/prometheus
$ chown prometheus:prometheus /var/lib/prometheus
#
# download prometheus
$ cd /opt/ && wget https://github.com/prometheus/prometheus/releases/download/v2.37.2/prometheus-2.37.2.linux-amd64.tar.gz
$ tar -xf prometheus-2.37.2.linux-amd64.tar.gz
```

## Configure Prometheus to let it know about the HTTP endpoints it should monitor

```bash
$ cd prometheus-2.37.2.linux-amd64
$ sudo cp /opt/prometheus-2.37.2.linux-amd64/prometheus /usr/local/bin
$ sudo cp /opt/prometheus-2.37.2.linux-amd64/promtool /usr/local/bin/
$ sudo chown prometheus:prometheus /usr/local/bin/prometheus
$ sudo chown prometheus:prometheus /usr/local/bin/promtool
#
# copy Prometheus Console Libraries
$ cp -r /opt/prometheus-2.37.2.linux-amd64/consoles /etc/prometheus
$ cp -r /opt/prometheus-2.37.2.linux-amd64/console_libraries /etc/prometheus
$ cp -r /opt/prometheus-2.37.2.linux-amd64/prometheus.yml /etc/prometheus
#
$ chown -R prometheus:prometheus /etc/prometheus/consoles
$ chown -R prometheus:prometheus /etc/prometheus/console_libraries
$ chown -R prometheus:prometheus /etc/prometheus/prometheus.yml
# confirm installation
$ prometheus --version
$ promtool --version
$ cat /etc/prometheus/prometheus.yml
```

## Creating Prometheus Systemd file

```bash
$ nano /etc/systemd/system/prometheus.service
## add
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
$ systemctl daemon-reload
$ systemctl enable prometheus
$ systemctl start prometheus && systemctl status prometheus
```

Add port 9090 to firewall whitelist. You can access at http://<your_server_IP>:9090/

## Setup Node Exporter 

```bash
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz
$ tar -xf node_exporter-1.4.0.linux-amd64.tar.gz
$ sudo mv node_exporter-1.4.0.linux-amd64/node_exporter /usr/local/bin
$ rm -r node_exporter-1.4.0.linux-amd64*
# -r flag to indicate it is a system account, and set the default shell to /bin/false using -s to prevent logins
$ sudo useradd -rs /bin/false node_exporter
# create a systemd unit file so that node_exporter can be started at boot
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

# restart service
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

## Configure the Node Exporter as a Prometheus target

```bash
$ cd /etc/prometheus
$ nano prometheus.yml
# Now in static_configs in your configuration file replace the target line with the below one
 # A scrape configuration containing exactly one endpoint to scrape from node_exporter running on a host:
 scrape_configs:
     # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
     - job_name: 'node'

     # metrics_path defaults to '/metrics'
     # scheme defaults to 'http'.

     static_configs:
     - targets: ['localhost:9090, localhost:9100']
     
# restart the service
$ systemctl restart prometheus

## Install Grafana

```bash
$ wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
$ echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
$ apt-get update
$ apt-get install grafana
$ systemctl start grafana-server && systemctl status grafana-server
$ systemctl enable grafana-server.service
```

## Create an override file to serve Grafana on a port < 1024

```bash
# Alternatively, create a file in /etc/systemd/system/grafana-server.service.d/override.conf
$ systemctl edit grafana-server.service
```

Add these additional settings to grant the **CAP_NET_BIND_SERVICE** capability. https://man7.org/linux/man-pages/man7/capabilities.7.html

```
[Service]
# Give the CAP_NET_BIND_SERVICE capability
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# A private user cannot have process capabilities on the host's user
# namespace and thus CAP_NET_BIND_SERVICE has no effect.
PrivateUsers=false
```

Confirm this:

```bash
$ nano /etc/grafana/grafana.ini
#
# The HTTP port  to use
;http_port = 1024 #3000
```

Start the server with init.d

```bash
$ service grafana-server start
$ service grafana-server status
$ update-rc.d grafana-server defaults
```

## Configure Prometheus as Grafana DataSource

Once you logged into Grafana Now first Navigate to **Settings Icon** ->> **Configuration** ->> **data sources**
