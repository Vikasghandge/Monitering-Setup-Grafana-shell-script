# Monitoring-Setup-Grafana-shell-script

Automated scripts to install and configure **Prometheus**, **Grafana**, and **node_exporter** so you can quickly start monitoring your servers.

---

## Overview

This repository contains three shell scripts (one for each component) that automate the common installation and configuration steps for a simple monitoring stack. The scripts are intended to be run on Linux servers (Ubuntu/CentOS/RHEL-compatible) and assume you have `sudo` privileges.

**High-level flow (recommended):**

1. Install `node_exporter` on every server you want to monitor (targets).
2. Install `prometheus` on your monitoring server and add the node_exporter targets.
3. Install `grafana` on the same prometheus  server (or a separate server) and connect it to Prometheus as a data source.

Ports used (defaults):

* `node_exporter`: **9100**
* `prometheus`: **9090**
* `grafana`: **3000**

---

## What's included (scripts)

* `setup-node-exporter.sh` — install & configure node_exporter as a systemd service on a target node.
* `setup-prometheus.sh` — install Prometheus, create directories and a `prometheus.yml` with targets (the script prompts for target IP:PORT and adds it to Prometheus targets).
* `setup-grafana.sh` — install Grafana, start the service and (optionally) create a Prometheus data source and import dashboards depending on the script's implementation.

> If your actual filenames are different, replace them below with your script names.

---

## Prerequisites

* A Linux server with network access to the nodes you want to monitor.
* `sudo` or root access on the machines where you run the scripts.
* Basic tools installed: `curl`, `wget`, `tar`, `systemctl`.
* Ensure ports 9100, 9090, 3000 are reachable between servers as appropriate (firewall rules, security groups).

---

## Quick start — step by step (simple)

1. **Install node_exporter on each target server** (the servers you want to monitor):

```bash
# on each target node (example filename)
chmod +x setup-node-exporter.sh
sudo ./setup-node-exporter.sh
```

The script will:

* Download `node_exporter` binary
* Create a `node_exporter` user
* Install a systemd unit file for node_exporter
* Enable & start the node_exporter service

After the script finishes the exporter will be available at `http://<NODE_IP>:9100/metrics`.

2. **Install Prometheus on the monitoring server** (this is the server that scrapes the node_exporters):

```bash
chmod +x setup-prometheus.sh
sudo ./setup-prometheus.sh
```

The `setup-prometheus.sh` script will prompt you for the target(s) to add into Prometheus, e.g. `192.168.0.10:9100`.

What the script will do (typical):

* Download Prometheus binaries and set up directories (e.g. `/etc/prometheus`, `/var/lib/prometheus`)
* Create a `prometheus` systemd service
* Write a `prometheus.yml` that includes the node_exporter target(s)
* Enable & start Prometheus

3. **Install Grafana on the monitoring server (or another server)**:

```bash
chmod +x setup-grafana.sh
sudo ./setup-grafana.sh
```

The script will typically:

* Install Grafana package or binary
* Start and enable the `grafana-server` service

After install, open Grafana in your browser at `http://<GRAFANA_IP>:3000` (default credential: `admin` / `admin`) — change the password immediately.

4. **Connect Grafana to Prometheus**

In Grafana UI:

* Go to **Configuration → Data Sources → Add data source → Prometheus**
* Set URL to `http://<PROMETHEUS_IP>:9090` and click **Save & Test**

5. **Import dashboards**

* Use the Grafana **Import** feature to add dashboards (there are ready-made Node Exporter dashboards available in the Grafana community dashboards).

---

## Detailed: What each script does & purpose

### `setup-node-exporter.sh` — Purpose

Purpose: Install and run the Prometheus Node Exporter on a machine so it exposes host-level metrics (CPU, memory, disk, network) that Prometheus can scrape.

Typical steps the script performs:

1. Download the latest compatible `node_exporter` binary.
2. Create a dedicated `node_exporter` (non-login) user.
3. Place the `node_exporter` binary in `/usr/local/bin` (or similar).
4. Create and install a `systemd` unit file: `/etc/systemd/system/node_exporter.service`.
5. `systemctl daemon-reload && systemctl enable --now node_exporter`.
6. Verify that `http://localhost:9100/metrics` returns data.

**Tip:** If you are using firewalls or cloud security groups, open TCP port **9100** for the Prometheus server.

### `setup-prometheus.sh` — Purpose

Purpose: Install Prometheus server, create a working `prometheus.yml` and start scraping the node_exporter targets.

Typical steps the script performs:

1. Download Prometheus official release and extract.
2. Create the `prometheus` user and directories `/etc/prometheus` and `/var/lib/prometheus`.
3. Create or update `/etc/prometheus/prometheus.yml` adding a `job_name` (for example `node_exporter`) under `scrape_configs` and include the target(s) you provide when prompted (example target: `192.168.0.10:9100`).
4. Install a `systemd` unit for `prometheus` and start it.

**Example `prometheus.yml` snippet**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.0.10:9100', '192.168.0.11:9100']
```

To add more targets later, you can either re-run the script if it supports adding targets, or manually edit `/etc/prometheus/prometheus.yml` and restart Prometheus:

```bash
# after editing prometheus.yml
sudo systemctl restart prometheus
# check status
sudo systemctl status prometheus
```

You can confirm Prometheus is scraping targets at: `http://<PROMETHEUS_IP>:9090/targets` (Prometheus UI → Status → Targets).

### `setup-grafana.sh` — Purpose

Purpose: Install Grafana, enable the service and optionally import a Prometheus data source & dashboards so you can visualize metrics collected by Prometheus.

Typical steps the script performs:

1. Install Grafana (package or download).
2. Start and enable the `grafana-server` service.
3. Optionally create a data source pointing to `http://<PROMETHEUS_IP>:9090` using Grafana provisioning or Grafana HTTP API.
4. (Optional) Import pre-defined dashboards (via provisioning or API).

After Grafana is running, visit `http://<GRAFANA_IP>:3000` and set up the Prometheus data source if not already auto-provisioned.

---

## Verify & test

* **Check node_exporter on target**:

  * `curl http://<NODE_IP>:9100/metrics` — should return metric lines.
* **Check Prometheus**:

  * Visit `http://<PROMETHEUS_IP>:9090/targets` and confirm your node targets are `UP`.
  * Use the Prometheus expression browser `http://<PROMETHEUS_IP>:9090/graph` to query metrics like `node_cpu_seconds_total`.
* **Check Grafana**:

  * Visit `http://<GRAFANA_IP>:3000` and verify you can query Prometheus via Explore tab or dashboards.

---

## Troubleshooting

* **Targets are DOWN in Prometheus**:

  * Check that node_exporter is running on the node: `systemctl status node_exporter`.
  * Ensure `node_exporter` listens on 0.0.0.0:9100 or the monitoring server can reach it over the network.
  * Disable firewall or add rule: `sudo ufw allow 9100/tcp` (Ubuntu) or adjust cloud security group rules.

* **Prometheus not starting**:

  * Check logs: `sudo journalctl -u prometheus -f`.
  * Validate `prometheus.yml` for YAML syntax mistakes.

* **Grafana login fails**:

  * Default credentials are usually `admin` / `admin` — change on first login.
  * Check Grafana logs: `sudo journalctl -u grafana-server -f`.

---

## Security & best practices

* Do not expose Prometheus or Grafana to the public internet without authentication and TLS.
* Use HTTPS / reverse proxy (nginx) with basic auth or OAuth for Grafana.
* Restrict access to node_exporter to only the Prometheus server (via firewall rules or private network).
* Rotate Grafana admin credentials and use strong passwords.

---

## Example repository layout (suggested)

```
Monitoring-Setup-Grafana-shell-script/
├── README.md                 # this file
├── setup-node-exporter.sh    # installs node_exporter on target nodes
├── setup-prometheus.sh       # installs prometheus and adds targets
├── setup-grafana.sh         # installs grafana and configures datasource/dashboards

```

---

## How to add more monitored nodes

1. Run `setup-node-exporter.sh` on each new node.
2. Add the node IP:PORT into Prometheus' `prometheus.yml` (or re-run the `add-target-prometheus.sh` script if it supports re-adding targets), then restart Prometheus.

---

## Uninstall / Cleanup (basic idea)

* Stop & disable services:

  * `sudo systemctl stop node_exporter && sudo systemctl disable node_exporter`
  * `sudo systemctl stop prometheus && sudo systemctl disable prometheus`
  * `sudo systemctl stop grafana-server && sudo systemctl disable grafana-server`
* Remove binary files and systemd unit files you created.

---



