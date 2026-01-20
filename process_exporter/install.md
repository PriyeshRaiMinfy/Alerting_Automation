# Process Exporter Installation
## Step 7: Download and Install

bash

''' 
cd /tmp
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.7.10/process-exporter-0.7.10.linux-amd64.tar.gz
tar xzf process-exporter-0.7.10.linux-amd64.tar.gz
sudo mv process-exporter-0.7.10.linux-amd64/process-exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/process-exporter

'''


## Step 8: Configuration
bash
sudo mkdir -p /etc/process-exporter
sudo tee /etc/process-exporter/processes.yml << 'EOF'
process_groups:
  - job_name: 'demo-services'
    static_configs:
      - labels:
          groupname: 'python_worker'
          app: 'data-processor'
        cmdline:
        - 'python3.*worker.py'
  - job_name: 'demo-services'
    static_configs:
      - labels:
          groupname: 'java_billing'
          app: 'billing-service'
        cmdline:
        - 'java.*BillingApp'
EOF
Step 9: Systemd Service
bash
sudo tee /etc/systemd/system/process-exporter.service << 'EOF'
[Unit]
Description=Process Exporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/process-exporter \
  --config.path=/etc/process-exporter/processes.yml \
  --web.listen-address=0.0.0.0:9256 \
  --log.level=info
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
Step 10: Start Process Exporter
bash
sudo systemctl daemon-reload
sudo systemctl enable --now process-exporter
sudo systemctl status process-exporter