# SurrealDB Production Cluster Deployment Guide

This guide outlines the deployment of a production-ready SurrealDB cluster using Ansible, with TiKV as the backing store.

## Prerequisites

### Hardware Requirements

#### TiKV Node (1 VM)
- CPU: 8+ cores
- RAM: 16GB minimum (32GB recommended)
- Storage: 200GB+ SSD (NVMe preferred)
- Network: 1Gbps minimum

#### SurrealDB Nodes (3 VMs)
- CPU: 4+ cores
- RAM: 8GB minimum (16GB recommended)
- Storage: 50GB+ SSD
- Network: 1Gbps minimum

### Software Requirements
- Ubuntu 20.04 LTS or later
- Python 3.8+
- Ansible 2.9+
- SSH access to all nodes

### Network Requirements
- Open ports:
  - TiKV: 2379 (PD)
  - SurrealDB: 8001, 8002, 8003
  - SSH: 22
- All nodes must be able to communicate with each other
- Stable network with low latency between nodes

## Directory Structure

```
.
├── README.md
├── ansible.cfg
├── inventory/
│   ├── production.yml
│   └── group_vars/
│       ├── all.yml
│       └── all/
│           └── vault.yml
├── roles/
│   ├── common/
│   ├── tikv/
│   ├── surrealdb/
│   └── haproxy/
└── site.yml
```

## Deployment Steps

1. **Clone the Repository**
```bash
git clone <your-repo>
cd surrealdb-cluster
```

2. **Configure Inventory**
Edit `inventory/production.yml`:
```yaml
pd1:
  ansible_host: <TIKV_VM_IP>
primary:
  ansible_host: <PRIMARY_DB_IP>
secondary1:
  ansible_host: <SECONDARY1_DB_IP>
secondary2:
  ansible_host: <SECONDARY2_DB_IP>
```

3. **Set Up SSH Access**
```bash
# Generate SSH key if needed
ssh-keygen -t rsa -b 4096

# Copy to all servers
for ip in <TIKV_VM_IP> <PRIMARY_DB_IP> <SECONDARY1_DB_IP> <SECONDARY2_DB_IP>; do
    ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@$ip
done
```

4. **Create Vault Password**
```bash
# Create a secure password file
echo "your-secure-vault-password" > .vault_pass
chmod 600 .vault_pass

# Create encrypted vars
ansible-vault create inventory/group_vars/all/vault.yml
```

Add to vault.yml:
```yaml
vault_surreal_password: "YourSecureDBPassword123!"
```

5. **Deploy the Cluster**
```bash
# Test connectivity
ansible all -m ping

# Deploy
ansible-playbook site.yml --vault-password-file .vault_pass
```

## Verification

1. **Check TiKV Status**
```bash
curl http://<TIKV_VM_IP>:2379/pd/api/v1/members
```

2. **Test SurrealDB Nodes**
```bash
# Connect to primary
surreal sql --conn http://<PRIMARY_DB_IP>:8001 --user root --pass YourSecureDBPassword123!

# Create test record
CREATE person:test SET name = 'Test User';

# Verify on secondaries
surreal sql --conn http://<SECONDARY1_DB_IP>:8002 --user root --pass YourSecureDBPassword123!
SELECT * FROM person:test;
```

## Monitoring

1. **Log Locations**
```bash
# TiKV logs
/data/tikv/logs/

# SurrealDB logs
/var/log/surrealdb/surrealdb.log
```

2. **Health Checks**
```bash
# Check TiKV cluster status
curl http://<TIKV_VM_IP>:2379/pd/api/v1/health

# Check SurrealDB nodes
for port in 8001 8002 8003; do
    curl -I http://localhost:$port/health
done
```

## Backup and Recovery

1. **TiKV Backup**
```bash
# Using TiKV BR tool
tiup br backup --pd <TIKV_VM_IP>:2379 --storage s3://backup-bucket/
```

2. **Configuration Backup**
```bash
# Backup all configuration files
tar czf config-backup.tar.gz /etc/surrealdb/ /etc/tikv/
```

## Scaling

### Adding a New Secondary Node

1. Add the new node to inventory/production.yml
2. Run the playbook with --limit new_node
```bash
ansible-playbook site.yml --limit new_secondary --vault-password-file .vault_pass
```

## Troubleshooting

### Common Issues

1. **Node Connection Issues**
```bash
# Check network connectivity
ping <node_ip>

# Verify ports are open
nc -zv <node_ip> <port>
```

2. **TiKV Issues**
```bash
# Check PD leader
curl http://<TIKV_VM_IP>:2379/pd/api/v1/leader

# Check store status
curl http://<TIKV_VM_IP>:2379/pd/api/v1/stores
```

3. **SurrealDB Issues**
```bash
# Check logs
tail -f /var/log/surrealdb/surrealdb.log

# Verify process
systemctl status surrealdb
```

## Security Considerations

1. **Firewall Rules**
```bash
# Allow only necessary ports
ufw allow 2379/tcp  # TiKV
ufw allow 8001/tcp  # SurrealDB Primary
ufw allow 8002/tcp  # SurrealDB Secondary1
ufw allow 8003/tcp  # SurrealDB Secondary2
```

2. **SSL/TLS Configuration**
- Use SSL certificates for production
- Configure secure communication between nodes
- Enable encryption at rest

## Maintenance

### Regular Tasks

1. **Log Rotation**
- Logs are automatically rotated using logrotate
- Check `/etc/logrotate.d/surrealdb`

2. **Updates**
```bash
# Update TiKV
tiup update --self
tiup update --all

# Update SurrealDB
# Follow official release notes
```

3. **Monitoring**
- Set up Prometheus + Grafana
- Monitor system metrics
- Set up alerts for:
  - Disk usage > 80%
  - High CPU usage
  - Memory pressure
  - Network latency

## Support and Resources

- [SurrealDB Documentation](https://surrealdb.com/docs)
- [TiKV Documentation](https://tikv.org/docs)
- [GitHub Issues](https://github.com/surrealdb/surrealdb/issues)
- [Community Discord](https://discord.gg/surrealdb)