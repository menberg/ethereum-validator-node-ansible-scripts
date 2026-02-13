# Ethereum Validator Node Ansible Scripts for Rocket Pool

This Ansible automation suite provides a complete, production-ready setup for a Rocket Pool Smartnode setup with Docker hybrid mode using own Reth (Execution Layer) and Nimbus (Consensus Layer) clients.

## Credits

Thanks to Marge (chrochetnode.com) for advice and review on validator architecture and operations.

## Features

- **Complete Node Setup**: Automated installation and configuration of Ethereum validator infrastructure for Rocket Pool node staking.
- **Security First**: SSH hardening, firewall configuration, and secure user management
- **Production Ready**: System monitoring, logging, and service management
- **Modular Design**: Individual playbooks for different setup phases
- **Network Flexibility**: Configurable for different Ethereum networks (Hoodi testnet, Mainnet, etc.)

## Components

### Execution Layer (EL)

- **Reth**: High-performance Ethereum execution client
- Full node with RPC endpoints
- JWT authentication for secure CL-EL communication

### Consensus Layer (CL)

- **Nimbus**: Efficient Ethereum consensus client
- REST API and metrics endpoints

### System Infrastructure

- Automated OS configuration (Ubuntu/Debian)
- SSH hardening and firewall setup
- System user management and security
- Monitoring tools (glances, chrony for NTP)

## Prerequisites

- Ansible 2.9+ installed on control machine
- Target server with Ubuntu 20.04+ or Debian 11+
- Hardcoded for x86_64 host architecture
- SSH access to target server
- At least 2TB SSD storage recommended for full node

## Quick Start

### 1. Clone and Configure

```bash
git clone <repository-url>
cd validator-node-ansible-scripts
```

### 2. Setup Inventory

Edit `inventory/hosts.yml` with your server details:

```yaml
all:
  children:
    servers:
      hosts:
        ethnode:
          project_name: ethnode
          ansible_host: YOUR_SERVER_IP
          ansible_user: automation_admin
          ansible_ssh_private_key_file: "~/.ssh/your_key"
          ansible_port: 19497
```

### 3. Configure Variables

Copy and edit configuration files:

```bash
cp vars/config_untracked.template.yml vars/config_untracked.yml
# Edit vars/config_untracked.yml with your specific values
```

**Required configurations in `vars/config_untracked.yml`:**

- `fee_recipient_wallet_address`: Your validator fee recipient address

### 4. Run Setup Playbooks

Execute playbooks in order:

```bash
# 1. Initial OS setup
# (temporarily set "ansible_user" in hosts.yml to "{{ system_user }}")
ansible-playbook -i inventory/hosts.yml --ask-pass --ask-become-pass 01_setup_os_initial.yml

# 2. Create automation admin user
# (temporarily set "ansible_user" in hosts.yml to "{{ system_user }}")
ansible-playbook -i inventory/hosts.yml --ask-pass --ask-become-pass 02_create_user_automation_admin.yml

# 3. Setup firewall and SSH
ansible-playbook -i inventory/hosts.yml 03_setup_os_firewall_ssh.yml

# 4. Create JWT secret
ansible-playbook -i inventory/hosts.yml 04_create_jwt_secret.yml

# 5. Install Reth (Execution Layer)
ansible-playbook -i inventory/hosts.yml 31_install_reth.yml

# 6. Configure Reth
ansible-playbook -i inventory/hosts.yml 34_copy_reth_config_files.yml

# 7. Install Nimbus (Consensus Layer)
ansible-playbook -i inventory/hosts.yml 11_install_nimbus.yml

# 8. Configure Nimbus
ansible-playbook -i inventory/hosts.yml 14_copy_nimbus_config_files.yml
```

## Playbook Overview

| Playbook                              | Purpose                              | Privilege Required |
| ------------------------------------- | ------------------------------------ | ------------------ |
| `01_setup_os_initial.yml`             | OS configuration, packages, timezone | Yes                |
| `02_create_user_automation_admin.yml` | Create automation admin user         | Yes                |
| `03_setup_os_firewall_ssh.yml`        | Firewall and SSH hardening           | Yes                |
| `04_create_jwt_secret.yml`            | Generate JWT secret for EL-CL auth   | Yes                |
| `11_install_nimbus.yml`               | Install Nimbus CL client             | Yes                |
| `14_copy_nimbus_config_files.yml`     | Configure Nimbus services            | Yes                |
| `15_update_nimbus.yml`                | Update Nimbus to latest version      | Yes                |
| `31_install_reth.yml`                 | Install Reth EL client               | Yes                |
| `34_copy_reth_config_files.yml`       | Configure Reth service               | Yes                |
| `36_update_reth.yml`                  | Update Reth to latest version        | Yes                |

## Configuration

### Network Selection

Edit `vars/config_project.yml` to change networks:

```yaml
network: "hoodi" # Options: "hoodi", "mainnet", etc.
```

### Client Versions

Leave version fields empty for latest releases, or pin specific versions:

```yaml
reth_version: "" # Empty = latest, or "v1.9.3"
```

### Storage Configuration

Default paths (can be modified in `vars/config_project.yml`):

```yaml
# Reth data directory
reth_home_dir: "/mnt/nvme/rethEL"
reth_data_dir: "/mnt/nvme/rethEL/data"

# Nimbus data directories
nimbus_cl_data_dir: "/opt/nimbusCL/data"
```

## Monitoring and Maintenance

### Service Management

```bash
# Check service status
sudo systemctl status rethEL
sudo systemctl status nimbusCL

# View logs
sudo journalctl -u rethEL -f
sudo journalctl -u nimbusCL -f
```

### System Monitoring

The setup includes `glances` for system monitoring:

```bash
glances
```

### Updates

Use the update playbooks to keep clients current:

```bash
# Update Nimbus
ansible-playbook -i inventory/hosts.yml 15_update_nimbus.yml

# Update Reth
ansible-playbook -i inventory/hosts.yml 36_update_reth.yml
```

## Security Considerations

- SSH access restricted to key-based authentication only
- Firewall configured with minimal required ports
- Dedicated system users for each service
- JWT authentication between EL and CL clients
- RPC and metrics are open locally for Docker compatibilty
- External RPC and metrics access is blocked by UFW

## Ports Used

| Service         | Port  | Protocol | Purpose               |
| --------------- | ----- | -------- | --------------------- |
| Reth RPC (HTTP) | 8545  | TCP      | JSON-RPC API          |
| Reth RPC (WS)   | 8546  | TCP      | WebSocket RPC         |
| Reth P2P        | 30303 | TCP/UDP  | Ethereum P2P          |
| Reth Discovery  | 9200  | UDP      | Node discovery        |
| Reth Auth-RPC   | 8551  | TCP      | CL-EL communication   |
| Reth Metrics    | 9001  | TCP      | Prometheus metrics    |
| Nimbus P2P      | 9000  | TCP/UDP  | Consensus P2P         |
| Nimbus REST     | 5052  | TCP      | REST API              |
| Nimbus Metrics  | 8008  | TCP      | Prometheus metrics    |
| SSH             | 19497 | TCP      | Administrative access |

## Troubleshooting

### Common Issues

1. **Connection refused**: Check firewall rules and SSH configuration
2. **Service fails to start**: Check logs with `journalctl -u <service>`
3. **Sync issues**: Verify network configuration and peer connections
4. **Storage space**: Monitor disk usage with `df -h`

### Log Locations

- Reth: `/var/log/reth/`
- Nimbus CL: `/var/log/nimbus/nimbusCL.jsonl`
- System: `/var/log/syslog`

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly on a testnet setup
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Disclaimer

This software is provided as-is for educational and operational purposes. Always test configurations on testnets before mainnet deployment. The authors are not responsible for any financial losses incurred through the use of this software.
