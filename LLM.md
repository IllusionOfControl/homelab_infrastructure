# LLM Instructions & Repository Context: Homelab Infrastructure

## 1. Project Overview & Scope
This repository provides declarative Infrastructure-as-Code (IaC) for a self-hosted **Homelab** managed with **Ansible** on **Proxmox Virtual Environment (PVE)**.

Primary functional areas:
- **Proxmox VE Bootstrap**: Node base configuration, disabling enterprise repositories, enabling `pve-no-subscription`, suppressing web UI subscription nag, installing system utilities.
- **Internal Routed Network (WAN -> Proxmox -> VMs)**: Configuration of an isolated internal bridge (`vmbr1`), enabling IPv4 forwarding (`sysctl`), and setting up NAT/Masquerading via iptables so Proxmox routes traffic between the external WAN and private internal VM subnets.
- **ZeroTier VPN**: Automated deployment of ZeroTier One daemon and network joining for secure remote administration.
- **Custom Error Placeholders (tarampampam/error-pages)**: Standalone HTTP daemon serving styled, responsive placeholder pages (502/503/504) whenever backend VMs or applications are offline or unreachable.
- **Caddy Reverse Proxy**: Automatic internal TLS certificate management (`local_certs` / `tls internal`), routing requests to Proxmox and VM services, and proxying backend connection errors to `tarampampam/error-pages`.
- **Local DNS (dnsmasq)**: Local authoritative and caching DNS resolver bound to local and internal interfaces, serving `.homelab` local records with upstream forwarding.
- **Fedora Cloud Virtual Machines**: Automated download of Fedora Cloud Base images, generation of Cloud-Init gold templates, and provisioning of Fedora VMs with customized CPU, RAM, disk, networking, and SSH keys.
- **Fedora Baseline & Disk Expansion**: Automated online expansion of root partition and filesystem (BTRFS/XFS/EXT4) to 100% of underlying virtual disk capacity, elimination of redundant background services (`ModemManager`, `Cockpit`, `Plymouth`, `PackageKit`), and DNF package manager optimization.

---

## 2. Architecture & Network Topology

```text
       [ External Home LAN / WAN ]
                    │
                    │ (e.g. 192.168.1.100 on vmbr0 or ZeroTier)
                    ▼
       ┌──────────────────────────────┐
       │       Proxmox VE Host        │
       │  - NAT Router / Masquerade   │
       │  - Caddy Reverse Proxy       │
       │  - error-pages (127.0.0.1)   │
       │  - dnsmasq DNS (10.10.10.1)  │
       │  - ZeroTier One Node         │
       └──────────────┬───────────────┘
                      │
                      │ Internal Bridge: vmbr1 (10.10.10.1/24)
                      ▼
         ┌────────────┴───────────┐
         │                        │
         ▼                        ▼
┌──────────────────┐     ┌──────────────────┐
│  fedora-vm-01    │     │  fedora-vm-02    │
│  10.10.10.11/24  │     │  10.10.10.12/24  │
│  GW: 10.10.10.1  │     │  GW: 10.10.10.1  │
│  DNS: 10.10.10.1 │     │  DNS: 10.10.10.1 │
└──────────────────┘     └──────────────────┘
```

---

## 3. Technology Stack
- **IaC Engine**: Ansible (`ansible-core` >= 2.15).
- **Primary Collections**:
  - `community.proxmox`: Proxmox management (`proxmox_kvm`, `proxmox_disk`, etc.).
  - `ansible.posix`: System tuning (`sysctl`), file synchronization, authorized keys.
  - `ansible.builtin`: Native modules (`deb822_repository`, `systemd`, `template`, `assert`, `stat`).
  - `community.general`: Extended system utilities.
- **Hypervisor**: Proxmox VE (PVE 8.x+ based on Debian).
- **Virtual Machines**: Fedora Cloud Base (BTRFS root, Cloud-Init, QEMU Guest Agent).
- **Core Services**: Caddy, tarampampam/error-pages, dnsmasq, ZeroTier One.

---

## 4. Repository Layout

```text
homelab_infrastructure/
├── ansible.cfg                    # Optimized Ansible defaults for homelab
├── requirements.yml               # Ansible Galaxy collections requirements
├── inventory/
│   ├── hosts.yml                  # Inventory groups (proxmox, dns_servers, caddy_servers, fedora_vms)
│   └── group_vars/
│       ├── all.yml                # Central configuration (IPs, subnets, DNS, ZeroTier, VMs, error-pages)
│       └── fedora_vms.yml         # SSH ProxyJump bastion settings for internal VMs
├── playbooks/
│   ├── site.yml                   # Master site orchestration playbook
│   ├── 01_proxmox_base.yml        # Proxmox node bootstrap
│   ├── 02_network.yml             # Internal NAT network (WAN -> Proxmox -> VMs)
│   ├── 03_zerotier.yml            # ZeroTier One VPN setup
│   ├── 04_caddy.yml               # Caddy reverse proxy & error-pages placeholder service
│   ├── 05_dns.yml                 # dnsmasq local resolver
│   ├── 06_fedora_vms.yml          # Fedora template creation and VM provisioning
│   ├── 07_fedora_baseline.yml     # Fedora VM service cleanup, DNF tuning & disk auto-expansion
│   └── destroy_fedora_vms.yml     # Destroy provisioned Fedora VMs (optional template/image purge)
├── roles/
│   ├── proxmox_bootstrap/         # Package mirrors, pve-no-subscription, nag removal
│   ├── proxmox_network/           # vmbr1 bridge, sysctl ip_forward, iptables masquerade
│   ├── zerotier/                  # ZeroTier deb822 repo, daemon, network join
│   ├── error_pages/               # tarampampam/error-pages standalone HTTP placeholder daemon
│   ├── caddy/                     # Caddy deb822 repo, Caddyfile template with handle_errors
│   ├── dns_server/                # dnsmasq local DNS, upstream forwarders, static records
│   ├── fedora_vms/                # Image download, gold template creation, VM deployment
│   └── fedora_baseline/           # Service cleanup, DNF tuning, online root partition/fs auto-expansion
├── .gitignore
├── LLM.md                         # Context and guidelines for AI assistants
└── README.md                      # Human-readable documentation and quickstart
```

---

## 5. Development Conventions & Guidelines

### 5.1. Language & Code Style
- **English Only**: All task names, comments, documentation, variable names, and commit messages MUST be written in technical English.
- Use `snake_case` for variable names, role names, and file names.
- Role variables MUST be prefixed with the role name (e.g. `proxmox_`, `dns_`, `zerotier_`, `caddy_`, `error_pages_`, `fedora_`).

### 5.2. Idempotency & Clean Code
- Every task must be idempotent.
- Never use `command` or `shell` when a native Ansible module is available.
- Relative role paths (`../roles/<role_name>`) are used in playbooks so they resolve correctly regardless of environment.

---

## 6. Common Workflows

```bash
# Syntax check all playbooks
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check

# Run error-pages and reverse proxy
ansible-playbook -i inventory/hosts.yml playbooks/site.yml -t error_pages,caddy

# Run complete deployment
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```
