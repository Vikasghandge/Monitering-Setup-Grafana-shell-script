```
root@ip-172-31-42-120:/home/ubuntu# cat setup_node_exporter.sh
#!/bin/bash
set -e

NODE_EXPORTER_VERSION="1.6.1"

echo "[INFO] Updating system..."
sudo apt update -y

echo "[INFO] Creating Node Exporter user..."
sudo useradd --system --no-create-home --shell /bin/false node_exporter || true

echo "[INFO] Downloading Node Exporter v${NODE_EXPORTER_VERSION}..."
wget -q https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-${NODE_EXPORTER_VERSION}*

echo "[INFO] Creating Node Exporter systemd service..."
cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.logind
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reexec
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

echo "[INFO] Opening port 9100..."
sudo ufw allow 9100/tcp || true

echo "[SUCCESS] Node Exporter setup complete!"
echo "➡ Node Exporter running at: http://$(curl -s ifconfig.me):9100/metrics"
```
