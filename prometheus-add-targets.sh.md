## this script is for the add new targets in the prometheus.yml
```
#!/bin/bash
set -e

PROM_CONFIG="/etc/prometheus/prometheus.yml"

if [ ! -f "$PROM_CONFIG" ]; then
    echo "[ERROR] Prometheus configuration file not found at $PROM_CONFIG"
    exit 1
fi

# Ask user for new targets
read -p "Do you want to add monitoring targets? [y/n]: " add_targets

if [[ "$add_targets" != "y" && "$add_targets" != "Y" ]]; then
    echo "[INFO] No targets added. Exiting."
    exit 0
fi

# Backup existing configuration
#sudo cp $PROM_CONFIG "${PROM_CONFIG}.bak.$(date +%F_%T)"
#echo "[INFO] Backup of prometheus.yml created."

targets=()
while true; do
    read -p "Enter target IP:PORT (leave empty to finish): " target
    [[ -z "$target" ]] && break
    targets+=("$target")
done

if [ ${#targets[@]} -eq 0 ]; then
    echo "[INFO] No targets entered. Exiting."
    exit 0
fi

# Check if job_name 'custom_targets' exists
if grep -q "job_name: 'custom_targets'" $PROM_CONFIG; then
    echo "[INFO] Appending targets to existing 'custom_targets' job..."
    # Find the line number of the job
    start_line=$(grep -n "job_name: 'custom_targets'" $PROM_CONFIG | cut -d: -f1)
    # Insert each new target under static_configs
    for t in "${targets[@]}"; do
        sudo sed -i "$((start_line+2)) a\      - targets: ['$t']" $PROM_CONFIG
        ((start_line++))
    done
else
    echo "[INFO] Creating new 'custom_targets' job..."
    echo "  - job_name: 'custom_targets'" | sudo tee -a $PROM_CONFIG
    echo "    static_configs:" | sudo tee -a $PROM_CONFIG
    for t in "${targets[@]}"; do
        echo "      - targets: ['$t']" | sudo tee -a $PROM_CONFIG
    done
fi

# Reload Prometheus configuration
curl -X POST http://localhost:9090/-/reload
echo "[SUCCESS] New targets added and Prometheus reloaded."
```
