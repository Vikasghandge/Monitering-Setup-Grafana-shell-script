## shell script for prometheus 
```
#!/bin/bash
set -e

# =========================
# Rollback function
# =========================
rollback() {
    echo
    echo "[ERROR] Something went wrong. Rolling back all changes..."

    # Stop Prometheus if running
    sudo systemctl stop prometheus.service || true
    sudo systemctl disable prometheus.service || true

    # Remove Prometheus files
    sudo rm -rf /usr/local/bin/prometheus /usr/local/bin/promtool
    sudo rm -rf /etc/prometheus /data

    # Remove systemd service
    sudo rm -f /etc/systemd/system/prometheus.service
    sudo systemctl daemon-reexec

    # Remove Prometheus user
    sudo userdel prometheus || true

    echo "[ROLLBACK COMPLETE] All changes have been reverted."
}
trap rollback ERR

# =========================
# Variables
# =========================
PROM_VERSION="2.47.1"
PROM_TAR="prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
PROM_URL="https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/${PROM_TAR}"

# =========================
# System update
# =========================
echo "[INFO] Updating system..."
sudo apt update -y && sudo apt upgrade -y

# =========================
# Create Prometheus user
# =========================
if ! id prometheus &>/dev/null; then
    echo "[INFO] Creating Prometheus system user..."
    sudo useradd --system --no-create-home --shell /bin/false prometheus
fi

# =========================
# Download and setup Prometheus
# =========================
echo "[INFO] Downloading Prometheus v${PROM_VERSION}..."
wget -q $PROM_URL
tar -xvf $PROM_TAR
sudo mkdir -p /data /etc/prometheus

cd prometheus-${PROM_VERSION}.linux-amd64/
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
cd ..
rm -rf prometheus-${PROM_VERSION}.linux-amd64/ $PROM_TAR

sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# =========================
# Systemd service
# =========================
echo "[INFO] Creating Prometheus systemd service..."
cat <<EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/data \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries \\
  --web.listen-address=0.0.0.0:9090 \\
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reexec
sudo systemctl enable prometheus
sudo systemctl start prometheus

# =========================
# Configure scrape targets
# =========================
echo "[INFO] Configure Prometheus scrape targets..."
read -p "Do you want to add monitoring targets? [y/n]: " add_targets

if [[ "$add_targets" == "y" || "$add_targets" == "Y" ]]; then
    targets=()
    while true; do
        read -p "Enter target IP:PORT (leave empty to finish): " target
        [[ -z "$target" ]] && break
        targets+=("$target")
    done

    if [ ${#targets[@]} -gt 0 ]; then
        echo "  - job_name: 'custom_targets'" | sudo tee -a /etc/prometheus/prometheus.yml
        echo "    static_configs:" | sudo tee -a /etc/prometheus/prometheus.yml
        for t in "${targets[@]}"; do
            echo "      - targets: ['$t']" | sudo tee -a /etc/prometheus/prometheus.yml
        done

        # Reload Prometheus configuration without restarting
        curl -X POST http://localhost:9090/-/reload
        echo "[INFO] Prometheus targets updated successfully!"
    fi
fi

# =========================
# Open Prometheus port
# =========================
sudo ufw allow 9090/tcp || true

echo "[SUCCESS] Prometheus installed and configured!"
echo "Access Prometheus: http://$(curl -s ifconfig.me):9090"
```
