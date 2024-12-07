

1. **Vault Passwords**
Create your vault file with these secure values:
```yaml
# inventory/group_vars/all/vault.yml
vault_surreal_password: "YourSecureRootPassword123!"  # SurrealDB root password
```

2. **Inventory Details**
```yaml
# inventory/production.yml
pd_servers:
  hosts:
    pd1:
      ansible_host: <VM1_IP>        # Replace with your PD server IP
      ansible_user: ubuntu          # Replace with your SSH user
      ansible_ssh_private_key_file: ~/.ssh/your_key.pem  # Your SSH key path

tikv_servers:
  hosts:
    kv1:
      ansible_host: <VM2_IP>        # Replace with your TiKV server IP
      ansible_user: ubuntu          # Replace with your SSH user
      ansible_ssh_private_key_file: ~/.ssh/your_key.pem

surrealdb_servers:
  hosts:
    surreal1:
      ansible_host: <VM3_IP>        # Replace with your SurrealDB server IP
      ansible_user: ubuntu          # Replace with your SSH user
      ansible_ssh_private_key_file: ~/.ssh/your_key.pem
```

3. **System Users**
```yaml
# inventory/group_vars/all.yml
tikv_user: "tikv"         # System user for TiKV
tikv_group: "tikv"        # System group for TiKV
surreal_user: "surreal"   # System user for SurrealDB
surreal_group: "surreal"  # System group for SurrealDB
```

4. **Port Configuration**
```yaml
# inventory/group_vars/all.yml
pd_port: 2379            # TiKV PD port
surreal_port: 8000       # SurrealDB port
```

5. **Directory Paths**
```yaml
# roles/tikv/defaults/main.yml
tikv_data_dir: "/data/tikv"  # TiKV data directory
pd_data_dir: "/data/pd"      # PD data directory

# roles/surrealdb/defaults/main.yml
surrealdb_log_dir: "/var/log/surrealdb"  # SurrealDB log directory
```

6. **Version Information**
```yaml
# roles/tikv/defaults/main.yml
tikv_version: "7.1.0"    # TiKV version

# roles/surrealdb/defaults/main.yml
surrealdb_version: "1.1.1"  # SurrealDB version
```

7. **System Requirements for Each VM**:
- RAM: Minimum 8GB (16GB recommended for production)
- CPU: 4 cores minimum
- Storage: 100GB+ SSD recommended
- OS: Ubuntu 20.04/22.04 or CentOS/RHEL 8+
- Network: All nodes must be able to communicate with each other
- Ports to open:
  - TiKV PD: 2379
  - SurrealDB: 8000
  - SSH: 22

8. **SSH Access**:
```bash
# Generate SSH key if you haven't already
ssh-keygen -t rsa -b 4096

# Copy SSH key to each server
ssh-copy-id -i ~/.ssh/id_rsa.pub user@<VM1_IP>
ssh-copy-id -i ~/.ssh/id_rsa.pub user@<VM2_IP>
ssh-copy-id -i ~/.ssh/id_rsa.pub user@<VM3_IP>
```

9. **Create Ansible Vault**:
```bash
# Create vault file
ansible-vault create inventory/group_vars/all/vault.yml

# Edit vault file
ansible-vault edit inventory/group_vars/all/vault.yml

# Remember vault password for deployment
```

10. **Deployment Command**:
```bash
# First time deployment
ansible-playbook site.yml --ask-vault-pass -i inventory/production.yml

# If you need to update configuration
ansible-playbook site.yml --ask-vault-pass -i inventory/production.yml --tags configure
```

11. **Verification Commands**:
```bash
# Check TiKV cluster status
curl http://<VM1_IP>:2379/pd/api/v1/members

# Connect to SurrealDB
surreal sql --conn http://<VM3_IP>:8000 --user root --pass YourSecureRootPassword123!

# Check logs
sudo tail -f /var/log/surrealdb/surrealdb.log
```
