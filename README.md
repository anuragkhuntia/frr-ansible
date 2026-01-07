# FRR Ansible Playbook

This Ansible playbook deploys and configures FRR (Free Range Routing) on Linux systems with BGP configuration.

## Features

- Installs FRR package
- Deploys templated FRR configuration
- Configures BGP with interface peering
- Supports customizable hostname, interfaces, and IP addresses
- Enables BFD (Bidirectional Forwarding Detection)

## Requirements

- Ansible 2.9 or higher
- Target systems running Debian/Ubuntu Linux
- SSH access to target hosts with sudo privileges
- FRR 10.3.1 (will be installed if not present)

## Directory Structure

```
.
├── .gitlab-ci.yml           # GitLab CI/CD pipeline
├── .yamllint.yml           # YAML linting configuration
├── .ansible-lint           # Ansible linting configuration
├── deploy_frr.yml          # Main playbook
├── inventory.ini           # Inventory file with multiple hosts
├── vars/
│   └── frr_vars.yml        # Variables for all hosts
└── templates/
    ├── frr.conf.j2         # FRR configuration template
    └── daemons.j2          # FRR daemons configuration
```

## Configuration Variables

All host configurations are defined in `vars/frr_vars.yml`:

```yaml
frr_hosts:
  z1scuti03:
    hostname: z1scuti03
    interface1: eno5
    interface2: eno7
    loopback_ip: 10.30.19.6
    bgp_asn: 65496
  
  z1scuti04:
    hostname: z1scuti04
    interface1: eno5
    interface2: eno7all hosts in the `app-servers` group:

```ini
[app-servers]
z1scuti03 ansible_host=10.30.19.6 ansible_user=ubuntu
z1scuti04 ansible_host=10.30.19.7 ansible_user=ubuntu
leaf01 ansible_host=10.30.20.10 ansible_user=ubuntu
```

All host configurations are centralized in `vars/frr_vars.yml`
leaf01 ansible_host=10.30.20.10 ansible_user=ubuntu

[spine_routers]
z1scuti03
z1scuti04

[leaf_routers]
leaf01
```

Each host has its own variable file in `host_vars/` directory.

### 2. Customize Variables

Each host has its own variable file in the `host_vars/` directory:

```yaml
# host_vars/z1scuti03.yml
frr_hostname: z1scuti03
frr_interface1: eno5
frr_interface2: eno7
fdit `vars/frr_vars.yml` and add/modify host configurations:

```yaml
frr_hosts:
  z1scuti03:
    hostname: z1scuti03
    interface1: eno5
    interface2: eno7
    loopback_ip: 10.30.19.6
    bgp_asn: 65496
  
  newhost:
    hostname: newhost
    interface1: eth0
    interface2: eth1
    loopback_ip: 10.30.19.20
    bgp_asn: 65496servers
ansible -i inventory.ini app-servers -m ping

# Deploy to all servers
ansible-playbook -i inventory.ini deploy_frr.yml

# Deploy to specific host
ansible-playbook -i inventory.ini deploy_frr.yml --limit z1scuti03

# Deploy to multiple hosts
ansible-playbook -i inventory.ini deploy_frr.yml --limit "z1scuti03,leaf01"

# Run in check mode (dry run)
ansible-playbook -i inventory.ini deploy_frr.yml --checkdeploy_frr.yml --limit z1scuti03 \
  -e "frr_loopback_ip=10.30.19.99"
```

### 4. Verify Configuration

After running the playbook, verify the FRR configuration:

```bash
# SSH to the router
ssh user@router1

# Check FRR status
sudo systemctl status frr

# Access FRR shell
sudo vtysh

# View running configuration
show running-config

# Check BGP status
show bgp summary

# Check BGP neighbors
show bgp neighbors
```

## FRR Configuration Details

The playbook configures:

- **Interfaces**: Two BGP peering interfaces with IPv6 RA enabled
- **Loopback**: Loopback interface with /32 IP
- **BGP**: 
  - Internal BGP peering via interfaces
  - BFD enabled for fast failover
  - Fast convergence enabled
  - Multipath AS-path relaxation
  - Aggressive timers (1s keepalive, 3s hold)
- **Redistribution**: Kernel and connected routes

## Customization

### Adding New Hosts

1. Add the host to `inventory.ini`:
```ini
[app-servers]
newhost ansible_host=10.30.19.20 ansible_user=ubuntu
```

2. Add host configuration to `vars/frr_vars.yml`:
```yaml
frr_hosts:
  # ... existing hosts ...
  
  newhost:
    hostname: newhost
    interface1: eth0
    interface2: eth1
    loopback_ip: 10.30.19.20
    bgp_asn: 65496
```

3. Deploy to the new host:
```bash
ansible-playbook -i inventory.ini deploy_frr.yml --limit newhost
```

### Modifying Existing Host Configuration

Simply edit the host's section in `vars/frr_vars.yml`:

```yaml
frr_hosts:
  z1scuti03:
    hostname: z1scuti03
    interface1: eno5     # Change interface name
    interface2: eno7
    loopback_ip: 10.30.19.99  # Change IP
    bgp_asn: 65500      # Change ASN
```

## GitLab CI/CD Pipeline

The project includes a `.gitlab-ci.yml` file with the following stages:

### Pipeline Stages

1. **validate** - Syntax and YAML validation
   - `syntax-check`: Validates Ansible playbook syntax
   - `yaml-lint`: Validates YAML formatting
   - `ansible-lint`: Checks Ansible best practices

2. **test** - Testing before deployment
   - `dry-run`: Runs playbook in check mode (manual)
   - `deploy-single-host`: Test deployment to specific host (manual)

3. **deploy-staging** - Deploy to staging environment
   - `deploy-staging`: Deploy to staging server (manual, develop/main branches)

4. **deploy-production** - Deploy to production
   - `deploy-production`: Deploy to all production servers (manual, main branch only)
   - `deploy-production-confirmed`: Deploy with tag confirmation (tags only)

### Required GitLab CI/CD Variables

Configure these in GitLab CI/CD settings (Settings → CI/CD → Variables):

| Variable | Description | Masked |
|----------|-------------|--------|
| `SSH_PRIVATE_KEY` | SSH private key for server access | Yes |

### Running the Pipeline

```bash
# Push to main branch
git add .
git commit -m "Update FRR configuration"
git push origin main

# Create a tag for production deployment
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

The pipeline will automatically run validation stages. Deployment stages require manual approval in GitLab UI.

### Manual Job Variables

For `deploy-single-host` job, set the `TARGET_HOST` variable:
- Go to CI/CD → Pipelines → Job
- Set `TARGET_HOST` to the hostname (e.g., `z1scuti03`)
- Run the job

## Backup

The playbook automatically creates backups of existing FRR configurations before applying changes. Backups are stored in `/etc/frr/` with timestamps.

## Troubleshooting

### Check FRR logs
```bash
sudo tail -f /var/log/frr/frr.log
```

### Restart FRR manually
```bash
sudo systemctl restart frr
```

### Verify configuration syntax
```bash
sudo vtysh --check
```

## License

MIT

## Author

Created for FRR network automation