# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  
  # =====================
  # STAGE 1: RAW SCRIPT INJECTION
  # =====================
  config.vm.provision "shell", inline: <<-'RAW_SCRIPT'
    # Create common.sh directly (bypassing file transfer)
    sudo mkdir -p /usr/local/lib/devops
    sudo bash -c 'cat > /usr/local/lib/devops/common.sh <<"EOF"
#!/bin/bash

# Logging functions
log_success() {
    echo -e "\033[0;32m[✓] $1\033[0m"
}

log_error() {
    echo -e "\033[0;31m[✗] $1\033[0m" >&2
    exit 1
}
EOF'
    sudo chmod 644 /usr/local/lib/devops/common.sh

    # Create loader
    echo 'source /usr/local/lib/devops/common.sh || exit 1' | sudo tee /etc/profile.d/devops-common.sh
    sudo chmod +x /etc/profile.d/devops-common.sh
  RAW_SCRIPT

  # =====================
  # STAGE 2: NETWORK SETUP
  # =====================
  config.vm.provision "shell", inline: <<-'SHELL'
    # Force reliable DNS
    sudo rm -f /etc/resolv.conf
    echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
    echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
    sudo chattr +i /etc/resolv.conf
    
    # Add host entries
    {
      echo "192.168.56.35 lb.devops-lab"
      echo "192.168.56.36 web1.devops-lab"
      echo "192.168.56.37 web2.devops-lab"
      echo "192.168.56.40 monitor.devops-lab"
    } | sudo tee -a /etc/hosts
  SHELL

  # =====================
  # LOAD BALANCER (35)
  # =====================
  config.vm.define "lb" do |lb|
    lb.vm.hostname = "lb.devops-lab"
    lb.vm.network "private_network", ip: "192.168.56.35"
    lb.vm.provision "shell", inline: <<-'LB_SCRIPT'
      source /usr/local/lib/devops/common.sh
      log_success "Starting LB setup"
      
      # Install Nginx
      apt-get update
      apt-get install -y nginx

      # Configure LB
      cat > /etc/nginx/nginx.conf <<'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server web1.devops-lab;
        server web2.devops-lab;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
}
EOF

      nginx -t && systemctl restart nginx
      log_success "LB setup complete"
    LB_SCRIPT
  end

  # =====================
  # WEB SERVERS (36,37)
  # =====================
  (1..2).each do |i|
    config.vm.define "web#{i}" do |web|
      web.vm.hostname = "web#{i}.devops-lab"
      web.vm.network "private_network", ip: "192.168.56.3#{i+5}" # 36,37
      web.vm.provision "shell", inline: <<-'WEB_SCRIPT'
        source /usr/local/lib/devops/common.sh
        log_success "Starting web server setup"

        # Install Nginx
        apt-get update
        apt-get install -y nginx wget

        # Set unique content
        echo "Hello from $(hostname)" > /var/www/html/index.html

        # Install Node Exporter
        wget -q https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
        tar xf node_exporter-*.tar.gz
        mv node_exporter-*/node_exporter /usr/local/bin/
        useradd -rs /bin/false node_exporter

        # Configure service
        cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Node Exporter
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
EOF

        systemctl daemon-reload
        systemctl enable --now node_exporter
        systemctl restart nginx
        log_success "Web server setup complete"
      WEB_SCRIPT
    end
  end

  # =====================
  # MONITORING (40)
  # =====================
  config.vm.define "monitor" do |mon|
    mon.vm.hostname = "monitor.devops-lab"
    mon.vm.network "private_network", ip: "192.168.56.40"
    mon.vm.provision "shell", inline: <<-'MON_SCRIPT'
      source /usr/local/lib/devops/common.sh
      log_success "Starting monitoring setup"

      # Install Prometheus
      wget -q https://github.com/prometheus/prometheus/releases/download/v2.30.3/prometheus-2.30.3.linux-amd64.tar.gz
      tar xf prometheus-*.tar.gz
      mv prometheus-*/prometheus /usr/local/bin/
      mkdir -p /etc/prometheus

      # Config
      cat > /etc/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['web1.devops-lab:9100', 'web2.devops-lab:9100']
EOF

      # Service
      cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus
[Service]
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml
[Install]
WantedBy=multi-user.target
EOF

      # Install Grafana
      apt-get install -y apt-transport-https software-properties-common
      wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
      echo "deb https://packages.grafana.com/oss/deb stable main" | tee /etc/apt/sources.list.d/grafana.list
      apt-get update
      apt-get install -y grafana

      # Start services
      systemctl daemon-reload
      systemctl enable --now prometheus
      systemctl enable --now grafana-server
      log_success "Monitoring setup complete"
    MON_SCRIPT
  end
end