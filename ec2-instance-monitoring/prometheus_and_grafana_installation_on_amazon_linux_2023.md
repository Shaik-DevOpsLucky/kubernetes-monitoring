# Prometheus and Grafana Installation & Setup on Amazon Linux 2023

This guide explains how to install and configure **Prometheus** and **Grafana** on **Amazon Linux 2023**, run Prometheus as a systemd service, and visualize metrics using Grafana dashboards.

---

## Prerequisites

- Amazon Linux 2023 EC2 instance
- Sudo/root access
- Required ports open in Security Group:
  - **9090** – Prometheus
  - **3000** – Grafana
  - **9100** – Node Exporter (targets)
  - **9117, 9121, 9104** – Exporters (Apache, Redis, MariaDB)

---

## Prometheus Installation

### 1. Create Prometheus User and Directories

```bash
sudo useradd --no-create-home prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

---

### 2. Download and Install Prometheus Binary

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.49.1/prometheus-2.49.1.linux-amd64.tar.gz

tar -xvf prometheus-2.49.1.linux-amd64.tar.gz

sudo cp prometheus-2.49.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.49.1.linux-amd64/promtool /usr/local/bin/
sudo cp -r prometheus-2.49.1.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-2.49.1.linux-amd64/console_libraries /etc/prometheus/
```

---

### 3. Prometheus Configuration

Create the configuration file:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

**prometheus.yml**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:

rule_files:

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "app1"
    static_configs:
      - targets: ["10.0.26.248:9100"]

  - job_name: "app2"
    static_configs:
      - targets: ["10.0.36.58:9100"]

  - job_name: "analytics"
    static_configs:
      - targets: ["10.0.50.231:9100"]

  - job_name: "DB1"
    static_configs:
      - targets: ["10.0.74.68:9100"]

  - job_name: "DB2"
    static_configs:
      - targets: ["10.0.91.253:9100"]

  - job_name: "app1-httpd"
    static_configs:
      - targets: ["10.0.26.248:9117"]

  - job_name: "app2-httpd"
    static_configs:
      - targets: ["10.0.36.58:9117"]

  - job_name: "analytics_redis"
    static_configs:
      - targets: ["10.0.50.231:9121"]

  - job_name: "db1_mariadb"
    static_configs:
      - targets: ["10.0.74.68:9104"]

  - job_name: "db2_mariadb"
    static_configs:
      - targets: ["10.0.91.253:9104"]
```

---

### 4. Create Prometheus Systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

---

### 5. Set Ownership and Permissions

```bash
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

---

### 6. Start and Enable Prometheus

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

### 7. Access Prometheus UI

Open your browser:

```
http://<EC2_PUBLIC_IP>:9090
```

---

## Grafana Installation

### 1. Update System Packages

```bash
sudo yum update -y
```

---

### 2. Add Grafana Repository

```bash
sudo nano /etc/yum.repos.d/grafana.repo
```

```ini
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

---

### 3. Install Grafana

```bash
sudo yum install grafana -y
```

---

### 4. Start and Enable Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service
```

---

### 5. Access Grafana UI

Open your browser:

```
http://<EC2_PUBLIC_IP>:3000
```

Default credentials:

- **Username:** admin
- **Password:** admin

You will be prompted to set a new password on first login.

---

## Configure Prometheus as Grafana Data Source

1. Login to Grafana
2. Go to **Settings → Data Sources**
3. Click **Add data source**
4. Select **Prometheus**
5. Set URL:
   ```
   http://localhost:9090
   ```
6. Click **Save & Test**

---

## Import Grafana Dashboard

1. Go to **Dashboards → Import**
2. Enter Dashboard ID:

```
1860
```

3. Select Prometheus data source
4. Click **Import**

This dashboard provides Node Exporter system metrics.



**Install node-exporter**\
\
**node-exporter.sh**\


wget [https://github.com/prometheus/node\_exporter/releases/download/v1.5.0/node\_exporter-1.5.0.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz)

tar -xf node\_exporter-1.5.0.linux-amd64.tar.gz

sudo mv node\_exporter-1.5.0.linux-amd64/node\_exporter  /usr/local/bin

rm -rv node\_exporter-1.5.0.linux-amd64\*

sudo useradd -rs /bin/false node\_exporter



sudo cat <\<EOF | sudo tee /etc/systemd/system/node\_exporter.service

[Unit]

Description=Node Exporter

After=network.target



[Service]

User=node\_exporter

Group=node\_exporter

Type=simple

ExecStart=/usr/local/bin/node\_exporter



[Install]

WantedBy=multi-user.target

EOF



sudo cat /etc/systemd/system/node\_exporter.service

sudo systemctl daemon-reload  && sudo systemctl enable node\_exporter

sudo systemctl start node\_exporter.service && sudo systemctl status node\_exporter.service --no-pager

## Conclusion

You have successfully:

- Installed and configured Prometheus on Amazon Linux 2023
- Configured Prometheus as a systemd service
- Installed Grafana and connected it to Prometheus
- Imported a ready-made monitoring dashboard

Happy Monitoring 🚀

