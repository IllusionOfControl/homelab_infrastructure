# Homelab Infrastructure as Code (Ansible & Proxmox)

Declarative homelab automation on **Proxmox Virtual Environment (PVE)** managed with **Ansible**.

## Key Features
- **Proxmox Bootstrap**: Disables enterprise repositories, configures `pve-no-subscription`, suppresses web UI subscription nag, installs essential server utilities.
- **Internal Routed Network (WAN -> Proxmox -> VMs)**: Creates a dedicated isolated internal bridge (`vmbr1`, `10.10.10.0/24`), enables kernel IPv4 forwarding, and configures NAT masquerading so Proxmox acts as the default gateway for internal VMs.
- **ZeroTier One**: Automated installation of the ZeroTier One daemon and network joining for secure remote mesh access.
- **Caddy Reverse Proxy**: Automatic internal TLS management (`local_certs` / `tls internal`) reverse-proxying `*.home.arpa` domains to the Proxmox Web UI and internal VM services.
- **DNS (dnsmasq)**: Local caching DNS resolver listening on loopback, WAN, and internal bridge interfaces, providing local resolution for homelab hostnames (`home.arpa` per RFC 8375) and upstream resolution.
- **Fedora Cloud Virtual Machines**: Downloads the official `Fedora Cloud Base` qcow2 image, generates a Cloud-Init Gold Template (`VMID 9000`), and clones two Fedora VMs (`10.10.10.11`, `10.10.10.12`) with injected SSH keys and QEMU Guest Agent.
- **Fedora Baseline & Disk Expansion**: Expands root partition and filesystem (BTRFS/XFS/EXT4) to 100% of maximum virtual disk capacity online, removes bloatware (`ModemManager`, `Cockpit`, `Plymouth`, `PackageKit`, `abrt`), disables delay-inducing services (`NetworkManager-wait-online`), and tunes DNF (`fastestmirror`, parallel downloads).

---

## Network Topology

```text
       [ External Home Router / WAN ]
                     │
                     │ (e.g. 192.168.1.100 on vmbr0)
                     ▼
       ┌───────────────────────────────┐
       │        Proxmox VE Host        │
       │  - NAT Router / Masquerade    │
       │  - Caddy Reverse Proxy        │
       │  - dnsmasq DNS (10.10.10.1)   │
       │  - ZeroTier One Node          │
       └──────────────┬────────────────┘
                      │
                      │ Internal Bridge: vmbr1 (10.10.10.1/24)
                      ▼
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐
│  fedora-vm-01    │      │  fedora-vm-02    │
│  10.10.10.11/24  │      │  10.10.10.12/24  │
│  GW: 10.10.10.1  │      │  GW: 10.10.10.1  │
│  DNS: 10.10.10.1 │      │  DNS: 10.10.10.1 │
└──────────────────┘      └──────────────────┘
```

---

## Repository Layout

```text
homelab_infrastructure/
├── ansible.cfg                    # Ansible configuration
├── requirements.yml               # Ansible Galaxy collections requirements
├── inventory/
│   ├── hosts.yml                  # Inventory hosts and groups
│   └── group_vars/
│       ├── all.yml                # Central configuration variables
│       └── fedora_vms.yml         # Bastion / ProxyJump settings for internal VMs
├── playbooks/
│   ├── site.yml                   # Master playbook
│   ├── 01_proxmox_base.yml        # Proxmox base configuration
│   ├── 02_network.yml             # Internal NAT network setup
│   ├── 03_zerotier.yml            # ZeroTier One VPN setup
│   ├── 04_caddy.yml               # Caddy reverse proxy setup
│   ├── 05_dns.yml                 # Local DNS server setup
│   ├── 06_fedora_vms.yml          # Fedora template and VM provisioning
│   └── 07_fedora_baseline.yml     # Fedora VM service cleanup and DNF tuning
├── roles/
│   ├── proxmox_bootstrap/
│   ├── proxmox_network/
│   ├── zerotier/
│   ├── caddy/
│   ├── dns_server/
│   ├── fedora_vms/
│   └── fedora_baseline/           # Service cleanup, DNF tuning, online root partition/fs auto-expansion
├── .gitignore
├── LLM.md                         # Context and guidelines for LLM agents
└── README.md
```

---

## Quick Start

### 1. Install Galaxy Requirements
```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Host and Variables
Edit:
- `inventory/hosts.yml`: Update your Proxmox server IP (`ansible_host`).
- `inventory/group_vars/all.yml`:
  - `proxmox_wan_interface`: Your external WAN bridge or NIC (default: `vmbr0`).
  - `proxmox_internal_gateway`: Internal Proxmox gateway IP (default: `10.10.10.1`).
  - `homelab_ssh_public_key`: Your personal SSH public key to inject into VMs.
  - `zerotier_network_id`: Your ZeroTier 16-character Network ID.

### 3. Syntax Verification
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check
```

### 4. Run Playbooks

**Deploy complete infrastructure:**
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

**Selective execution via tags:**
```bash
# Only Proxmox base setup
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t proxmox

# Only Internal Network (Bridge vmbr1 & NAT)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t network

# Only ZeroTier
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t zerotier

# Only Reverse Proxy & DNS
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t caddy,dns

# Only Fedora Cloud VMs
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t vms

# Only Fedora VMs baseline cleanup & optimization
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t baseline
```
