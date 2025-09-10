---
sidebar_position: 5
title: Virtual Machines & Bare Metal
---


This guide covers deploying Laminar directly on virtual machines or bare metal servers. While Kubernetes is recommended for production, direct deployment offers full control over the infrastructure and can be suitable for specific use cases.

## Overview

Direct deployment on VMs or bare metal servers is suitable for:
- On-premises installations with specific hardware requirements
- Environments where Kubernetes is not available
- Performance-critical deployments requiring hardware optimization
- Development and testing environments
- Edge deployments with resource constraints

## System Requirements

### Minimum Requirements (Single Node)
- **CPU**: 4 cores (x86_64 or ARM64)
- **Memory**: 8 GB RAM
- **Storage**: 50 GB SSD
- **OS**: Linux (Ubuntu 20.04+, RHEL 8+, Debian 11+)
- **Network**: 1 Gbps

### Recommended Production Requirements (Per Node)
- **CPU**: 16+ cores
- **Memory**: 64+ GB RAM
- **Storage**: 500+ GB NVMe SSD
- **OS**: Ubuntu 22.04 LTS or RHEL 9
- **Network**: 10 Gbps

### Software Prerequisites
```bash
sudo apt-get update
sudo apt-get install -y \
  curl \
  wget \
  tar \
  gzip \
  systemd \
  postgresql-client \
  openssl \
  ca-certificates

sudo apt-get install -y \
  htop \
  iotop \
  netstat \
  tmux \
  vim
```

## Installation Methods

### Method 1: Binary Installation

#### Download Binary

```bash
LAMINAR_VERSION=latest
wget https://github.com/laminar/laminar/releases/download/${LAMINAR_VERSION}/laminar-linux-amd64.tar.gz

wget https://github.com/laminar/laminar/releases/download/${LAMINAR_VERSION}/laminar-linux-arm64.tar.gz

tar -xzf laminar-linux-amd64.tar.gz
sudo mv laminar /usr/local/bin/
sudo chmod +x /usr/local/bin/laminar

laminar --version
```

#### Create System User

```bash
sudo useradd -r -s /bin/false laminar

sudo mkdir -p /etc/laminar
sudo mkdir -p /var/lib/laminar
sudo mkdir -p /var/log/laminar

sudo chown -R laminar:laminar /etc/laminar
sudo chown -R laminar:laminar /var/lib/laminar
sudo chown -R laminar:laminar /var/log/laminar
```

### Method 2: Docker Installation

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

docker pull ghcr.io/laminar/laminar:latest

docker run -d \
  --name laminar \
  --restart unless-stopped \
  -p 8000:8000 \
  -p 8001:8001 \
  -p 9000:9000 \
  -v /etc/laminar:/etc/laminar \
  -v /var/lib/laminar:/var/lib/laminar \
  ghcr.io/laminar/laminar:latest
```

### Method 3: Build from Source

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

git clone https://github.com/laminar/laminar.git
cd laminar

RUSTFLAGS='-C target-cpu=native' cargo build --release

sudo cp target/release/laminar /usr/local/bin/
sudo chmod +x /usr/local/bin/laminar
```

## Configuration

### Basic Configuration

Create `/etc/laminar/config.yaml`:

```yaml
server:
  host: 0.0.0.0
  api_port: 8001
  ui_port: 8000
  grpc_port: 9000
  metrics_port: 9090

database:
  type: postgres  # or sqlite
  host: localhost
  port: 5432
  database: laminar
  username: laminar
  password: ${DATABASE_PASSWORD}
  pool_size: 25

storage:
  checkpoint:
    type: filesystem  # or s3, gcs, azure
    path: /var/lib/laminar/checkpoints
  artifact:
    type: filesystem
    path: /var/lib/laminar/artifacts

cluster:
  mode: standalone  # or distributed
  node_id: node-1
  advertise_address: 192.168.1.10

resources:
  max_memory: 32GB
  max_cpu_cores: 16
  task_slots: 32

logging:
  level: INFO
  file: /var/log/laminar/laminar.log
  max_size: 100MB
  max_backups: 10
```

### Environment Variables

Create `/etc/laminar/laminar.env`:

```bash
DATABASE_PASSWORD=secure_password
DATABASE_TYPE=postgres
DATABASE_URL=postgresql://laminar:secure_password@localhost:5432/laminar

LAMINAR_CHECKPOINT_URL=file:///var/lib/laminar/checkpoints
LAMINAR_ARTIFACT_URL=file:///var/lib/laminar/artifacts

AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=us-east-1

LAMINAR_CLUSTER_MODE=standalone
LAMINAR_NODE_ID=node-1

LAMINAR_PARALLELISM=16
LAMINAR_MEMORY_LIMIT=32g
RUST_LOG=info,laminar=debug
```

## Database Setup

### PostgreSQL Installation

```bash
sudo apt-get install -y postgresql postgresql-contrib

sudo systemctl start postgresql
sudo systemctl enable postgresql

sudo -u postgres psql <<EOF
CREATE USER laminar WITH PASSWORD 'secure_password';
CREATE DATABASE laminar OWNER laminar;
GRANT ALL PRIVILEGES ON DATABASE laminar TO laminar;
EOF

laminar db migrate --database-url postgresql://laminar:secure_password@localhost:5432/laminar
```

### SQLite Setup (Development Only)

```bash
sudo mkdir -p /var/lib/laminar/db
sudo chown laminar:laminar /var/lib/laminar/db

laminar db init --database-type sqlite --database-path /var/lib/laminar/db/laminar.db
```

## Systemd Service

Create `/etc/systemd/system/laminar.service`:

```ini
[Unit]
Description=Laminar Stream Processing Platform
Documentation=https://docs.laminar.dev
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=notify
User=laminar
Group=laminar
WorkingDirectory=/var/lib/laminar
EnvironmentFile=/etc/laminar/laminar.env
ExecStartPre=/usr/local/bin/laminar db migrate
ExecStart=/usr/local/bin/laminar server --config /etc/laminar/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=laminar

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/laminar /var/log/laminar

LimitNOFILE=65536
LimitNPROC=32768
TasksMax=infinity

MemoryMax=48G
MemoryHigh=40G

[Install]
WantedBy=multi-user.target
```

### Start Service

```bash
sudo systemctl daemon-reload

sudo systemctl start laminar

sudo systemctl enable laminar

sudo systemctl status laminar

sudo journalctl -u laminar -f
```

## Distributed Deployment

### Architecture

For production deployments, use multiple nodes:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Control    │     │  Control    │     │  Control    │
│  Node 1     │────▶│  Node 2     │────▶│  Node 3     │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Worker     │     │  Worker     │     │  Worker     │
│  Node 1     │     │  Node 2     │     │  Node N     │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Control Node Configuration

`/etc/laminar/control-node.yaml`:

```yaml
cluster:
  mode: distributed
  role: control
  node_id: control-1
  advertise_address: 10.0.1.10
  
  # Cluster members
  peers:
    - control-1:10.0.1.10:9000
    - control-2:10.0.1.11:9000
    - control-3:10.0.1.12:9000
  
  # Leader election
  election:
    enabled: true
    timeout: 5s
    heartbeat: 1s

components:
  api_server: true
  scheduler: true
  web_ui: true
  worker: false
```

### Worker Node Configuration

`/etc/laminar/worker-node.yaml`:

```yaml
cluster:
  mode: distributed
  role: worker
  node_id: worker-1
  advertise_address: 10.0.2.10
  
  # Control plane endpoints
  control_plane:
    - 10.0.1.10:9000
    - 10.0.1.11:9000
    - 10.0.1.12:9000

components:
  api_server: false
  scheduler: false
  web_ui: false
  worker: true

worker:
  slots: 32
  memory: 60GB
  data_dir: /var/lib/laminar/worker
```

### Network Configuration

Configure firewall rules:

```bash
sudo ufw allow from 10.0.2.0/24 to any port 9000
sudo ufw allow from 10.0.3.0/24 to any port 8000,8001

sudo ufw allow from 10.0.1.0/24 to any port 6669

sudo ufw allow from 10.0.0.0/16 to any port 9000,6669
```

## Performance Tuning

### System Optimization

```bash
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

sudo tee -a /etc/sysctl.conf <<EOF
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 134217728
net.ipv4.tcp_wmem=4096 65536 134217728
EOF

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### CPU Optimization

```bash
sudo apt-get install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils

CPUAffinity=0-15  # Bind to specific cores
```

### Memory Configuration

```bash
echo "vm.nr_hugepages=16384" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

LAMINAR_USE_HUGEPAGES=true
```

## Monitoring

### Prometheus Metrics

```yaml
scrape_configs:
  - job_name: 'laminar'
    static_configs:
      - targets:
        - 'control-1:9090'
        - 'control-2:9090'
        - 'control-3:9090'
        - 'worker-1:9090'
        - 'worker-2:9090'
```

### Health Checks

```bash
cat > /usr/local/bin/laminar-health.sh <<'EOF'
#!/bin/bash
curl -f http://localhost:8001/health || exit 1
EOF

chmod +x /usr/local/bin/laminar-health.sh

(crontab -l 2>/dev/null; echo "*/1 * * * * /usr/local/bin/laminar-health.sh || systemctl restart laminar") | crontab -
```

## Backup & Recovery

### Backup Script

```bash
#!/bin/bash

BACKUP_DIR="/backup/laminar/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

pg_dump -h localhost -U laminar laminar | gzip > $BACKUP_DIR/database.sql.gz

tar -czf $BACKUP_DIR/config.tar.gz /etc/laminar

tar -czf $BACKUP_DIR/state.tar.gz /var/lib/laminar

aws s3 sync $BACKUP_DIR s3://backups/laminar/
```

### Recovery Procedure

```bash
sudo systemctl stop laminar

gunzip < /backup/database.sql.gz | psql -h localhost -U laminar laminar

tar -xzf /backup/config.tar.gz -C /

tar -xzf /backup/state.tar.gz -C /

sudo systemctl start laminar
```

## Troubleshooting

### Common Issues

**Service Won't Start**
```bash
journalctl -u laminar -n 100 --no-pager

laminar config validate --config /etc/laminar/config.yaml

ls -la /var/lib/laminar
ls -la /etc/laminar
```

**High Memory Usage**
```bash
ps aux | grep laminar
htop -p $(pgrep laminar)

echo "LAMINAR_MEMORY_LIMIT=24g" >> /etc/laminar/laminar.env
sudo systemctl restart laminar
```

**Network Issues**
```bash
ss -tlnp | grep laminar

nc -zv control-node 9000
curl http://control-node:8001/health
```

## Security Hardening

### TLS Configuration

```yaml
tls:
  enabled: true
  cert_file: /etc/laminar/certs/server.crt
  key_file: /etc/laminar/certs/server.key
  ca_file: /etc/laminar/certs/ca.crt
  client_auth: true
```

### Firewall Rules

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow ssh

sudo ufw allow from 10.0.0.0/8 to any port 8000,8001,9000,9090,6669

sudo ufw enable
```

## Best Practices

1. **Use configuration management** (Ansible, Puppet, Chef) for consistent deployments
2. **Monitor system resources** continuously
3. **Implement log rotation** to prevent disk fill
4. **Regular backups** of configuration and state
5. **Use dedicated storage** for checkpoints
6. **Network segmentation** between control and data planes
7. **Regular security updates** for OS and dependencies
8. **Document your deployment** architecture and procedures
9. **Test disaster recovery** procedures regularly
10. **Use monitoring and alerting** for proactive issue detection