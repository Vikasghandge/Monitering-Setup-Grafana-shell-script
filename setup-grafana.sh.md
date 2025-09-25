## this is Automated shell script which Configure the grafana.
```
#!/bin/bash
set -e

# =========================
# Rollback function
# =========================
rollback() {
    echo
    echo "[ERROR] Something went wrong. Rolling back all changes..."

    # Stop Grafana if running
    sudo systemctl stop grafana-server.service || true
    sudo systemctl disable grafana-server.service || true

    # Remove Grafana package
    sudo apt-get remove --purge -y grafana || true
    sudo apt-get autoremove -y

    # Remove repository and GPG key
    sudo rm -f /etc/apt/sources.list.d/grafana.list
    sudo apt-key del $(apt-key list | grep "Grafana" -B 1 | head -n1 | awk '{print $2}') || true

    echo "[ROLLBACK COMPLETE] All changes have been reverted."
}
trap rollback ERR

# =========================
# Update system
# =========================
echo "[INFO] Updating system..."
sudo apt-get update -y && sudo apt-get upgrade -y

# =========================
# Install dependencies
# =========================
echo "[INFO] Installing dependencies..."
sudo apt-get install -y apt-transport-https software-properties-common wget

# =========================
# Add Grafana repo and GPG key
# =========================
echo "[INFO] Adding Grafana repository and GPG key..."
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update -y

# =========================
# Install Grafana
# =========================
echo "[INFO] Installing Grafana..."
sudo apt-get install -y grafana

# =========================
# Enable and start Grafana service
# =========================
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# =========================
# Open Grafana port
# =========================
sudo ufw allow 3000/tcp || true

echo "[SUCCESS] Grafana installed and running!"
echo "Access Grafana Web UI: http://$(curl -s ifconfig.me):3000"
echo "Default login: admin / admin (change on first login)"
```
