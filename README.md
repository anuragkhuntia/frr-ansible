# FRR Ansible Playbook

This Ansible playbook deploys and configures FRR (Free Range Routing) on Linux systems with BGP configuration.

## Features

- Installs FRR package
- Deploys templated FRR configuration
- Configures BGP with interface peering
- **Auto-discovers network interfaces** (UP and not in bond)
- Supports customizable hostname, interfaces, and IP addresses
- Enables BFD (Bidirectional Forwarding Detection)
- GitLab CI/CD pipeline included

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
├── main.yml                # Main playbook (discovery + deployment)
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
    # interface1: eno5        # Optional: leave blank for auto-discovery
    # interface2: eno7        # Optional: leave blank for auto-discovery
    loopback_ip: 10.30.19.6
    bgp_asn: 65496
```

**Interface Selection:**
- **Auto-discovery** (recommended): Comment out or don't define `interface1`/`interface2`
- **Manual selection**: Explicitly define interface names
- Auto-discovery selects interfaces that are:
  - UP (active)
  - NOT part of a bond
  - NOT loopback
  - NO IP address assigned

## Usage

### 0. Discover Available Interfaces (Recommended First Step)

Before deployment, discover eligible interfaces on your servers:

```bash
# Discover interfaces on all servers
ansible-playbook -i inventory.ini main.yml --tags discover

# Discover interfaces on specific host
ansible-playbook -i inventory.ini main.yml --tags discover --limit z1scuti03
```

**Output shows:**
- All network interfaces
- Bonded interfaces (excluded)
- Eligible interfaces (UP, not bonded, no IP)
- Which interfaces will be auto-selected (first 2 eligible)

### 1. Update Inventory

The `inventory.ini` includes all hosts in the `app-servers` group:

```ini
[app-servers]
z1scuti03 ansible_host=10.30.19.6 ansible_user=ubuntu
z1scuti04 ansible_host=10.30.19.7 ansible_user=ubuntu
leaf01 ansible_host=10.30.20.10 ansible_user=ubuntu
```

### 2. Customize Variables

Edit `vars/frr_vars.yml`:

```yaml
frr_hosts:
  z1scuti03:
    hostname: z1scuti03
    # interface1: eno5         # Leave commented for auto-discovery
    # interface2: eno7
    loopback_ip: 10.30.19.6
    bgp_asn: 65496
  
  z1scuti04:
    hostname: z1scuti04
    interface1: eth0           # Explicitly specified
    interface2: eth1
    loopback_ip: 10.30.19.7
    bgp_asn: 65496
```

### 3. Run the Playbook

```bash
# Check connectivity to all servers
ansible -i inventory.ini app-servers -m ping

# Deploy to all servers (default behavior)
ansible-playbook -i inventory.ini main.yml

# Deploy to specific host
ansible-playbook -i inventory.ini main.yml --limit z1scuti03

# Deploy to multiple hosts
ansible-playbook -i inventory.ini main.yml --limit "z1scuti03,leaf01"

# Run in check mode (dry run)
ansible-playbook -i inventory.ini main.yml --check --diff

# Run with specific tags
ansible-playbook -i inventory.ini main.yml --tags deploy
ansible-playbook -i inventory.ini main.yml --tags discover
```

### 4. Verify Configuration

After running the playbook, verify the FRR configuration:

```bash
# SSH to the server
ssh ubuntu@10.30.19.6

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

## Auto-Discovery Details

### How Interface Discovery Works

1. **Gather Facts**: Collects all network interface information from target server
2. **Identify Bonded Interfaces**: Reads `/proc/net/bonding/*` to find interfaces in bonds
3. **Filter Eligible Interfaces**: Selects interfaces that are:
   - Active (UP state)
   - Not loopback (`lo`)
   - Not part of a bond
   - Not `bonding_masters`
   - No IPv4 address assigned
4. **Auto-select**: Uses first 2 eligible interfaces if not defined in vars

### Manual Override

You can always override auto-discovery by explicitly defining interfaces in `vars/frr_vars.yml`:

```yaml
frr_hosts:
  myhost:
    hostname: myhost
    interface1: eth2    # Explicitly use eth2
    interface2: eth3    # Explicitly use eth3
    loopback_ip: 10.30.19.10
    bgp_asn: 65496
```

## Customization

### Adding New Hosts

1. Add to `inventory.ini`:
```ini
[app-servers]
newhost ansible_host=10.30.19.20 ansible_user=ubuntu
```

2. Add to `vars/frr_vars.yml`:
```yaml
frr_hosts:
  newhost:
    hostname: newhost
    # interface1: auto-discover
    # interface2: auto-discover
    loopback_ip: 10.30.19.20
    bgp_asn: 65496
```

3. Deploy:
```bash
ansible-playbook -i inventory.ini main.yml --limit newhost
```

### Modifying Existing Host Configuration

Edit `vars/frr_vars.yml`:

```yaml
frr_hosts:
  z1scuti03:
    hostname: z1scuti03
    interface1: eno9       # Change interface
    loopback_ip: 10.30.19.99  # Change IP
    bgp_asn: 65500            # Change ASN
```

## GitLab CI/CD Pipeline

The project includes a `.gitlab-ci.yml` file with automated testing and deployment.

### Pipeline Stages

1. **validate** - Syntax and YAML validation
   - `syntax-check`: Validates Ansible playbook syntax
   - `yaml-lint`: Validates YAML formatting
   - `ansible-lint`: Checks Ansible best practices

2. **test** - Testing before deployment
   - `dry-run`: Runs playbook in check mode (manual)
   - `deploy-single-host`: Test deployment to specific host (manual)

3. **deploy-staging** - Deploy to staging environment
   - Deploys to staging server (manual trigger)
   - Available on `develop` and `main` branches

4. **deploy-production** - Deploy to production
   - Deploys to all production servers (manual trigger)
   - `deploy-production`: Runs on `main` branch
   - `deploy-production-confirmed`: Runs on git tags only

### Required GitLab CI/CD Variables

Configure in GitLab: Settings → CI/CD → Variables

| Variable | Description | Masked |
|----------|-------------|--------|
| `SSH_PRIVATE_KEY` | SSH private key for server access | Yes |

### Running the Pipeline

```bash
# Push to main branch
git add .
git commit -m "Update FRR configuration"
git push origin main

# Create a release tag for production
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

Validation runs automatically. Deployments require manual approval in GitLab UI.

### Manual Job Variables

For `deploy-single-host` job:
1. Go to CI/CD → Pipelines → Job
2. Set `TARGET_HOST` variable (e.g., `z1scuti03`)
3. Run the job

## Backup

The playbook automatically creates backups of existing FRR configurations before applying changes. Backups are stored in `/etc/frr/` with timestamps.

## Troubleshooting

### Check discovered interfaces
```bash
ansible-playbook -i inventory.ini main.yml --tags discover --limit z1scuti03
```

### View FRR logs
```bash
sudo tail -f /var/log/frr/frr.log
```

### Restart FRR service
```bash
sudo systemctl restart frr
```

### Verify configuration syntax
```bash
sudo vtysh --check
```

### Debug playbook
```bash
ansible-playbook -i inventory.ini main.yml --limit z1scuti03 -vvv
```

### Common Issues

**Insufficient interfaces**: If auto-discovery finds less than 2 interfaces, the playbook will fail. Solutions:
- Manually define interfaces in `vars/frr_vars.yml`
- Check interface status: `ip link show`
- Verify interfaces are UP: `ip link set <interface> up`

**Interface already in bond**: Auto-discovery excludes bonded interfaces. To use them:
- Explicitly define interface in vars
- Or remove from bond configuration

## License

MIT

## Author

Created for FRR network automation
