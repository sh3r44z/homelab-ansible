# Homelab Ansible — Server Setup Speedrun

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

Crazy that one command can turn a empty clean linux server into a fully configured home lab.

```bash
ansible-playbook -i inventory.ini site.yml --ask-become-pass
```

---

## What is ti

A fully automated server provisioning system that SSHes into a target machine and brings it to a known, consistent state regardless of whether that machine is brand new or has been running for months.

The important thing to understand before cloning this: Ansible is **idempotent**, which i just learnt the meaning of lol.  Run the playbook on a fresh server and it builds everything from scratch. Run it again on the same server and it checks everything is correct, changes nothing, and exits cleanly. Same command. Same result. Every time. So sick! 

---

## What it does

**Level 1 — Base hardening**
Locks down the server before anything else goes on it. Disables root SSH login, turns off password authentication so only key-based access works, enables UFW firewall with a default-deny policy, opens only the ports the stack needs, and installs fail2ban to automatically ban IPs that fail SSH login too many times.

**Level 2 — Docker**
Installs Docker if it isn't already present. If Docker is already installed it detects this and skips the installation entirely. Adds the user to the docker group and ensures the service starts on boot.

**Level 3 — Monitoring stack**
Deploys Prometheus, Grafana, Alertmanager, Node Exporter, cAdvisor and Uptime Kuma via Docker Compose. Config files are managed by Ansible so the stack is fully reproducible.

**Level 4 — Automation**
Deploys n8n for workflow automation.Ansible manages the compose file and ensures the service is running.

---

## Project structure

```
homelab-ansible/
├── site.yml                    Entry point (runs all roles in sequence)
├── inventory.ini               Target server(s) and connection config
└── roles/
    ├── base/
    │   ├── tasks/main.yml      Hardening, firewall, SSH config
    │   └── handlers/main.yml   SSH restart handler
    ├── docker/
    │   └── tasks/main.yml      Docker install with idempotency check
    ├── monitoring/
    │   ├── tasks/main.yml      Monitoring stack deployment
    │   └── files/              docker-compose.yml, prometheus.yml, alerts.yml, alertmanager.yml
    └── automation/
        ├── tasks/main.yml      n8n deployment
        └── files/              docker-compose.yml
```

---

## Prerequisites

- Ansible installed on your control machine (`brew install ansible` on Mac)
- SSH key-based access to the target server already configured
- The `community.docker` Ansible collection installed:

```bash
ansible-galaxy collection install community.docker
```

---

## Setup

**1. Clone the repo**
```bash
git clone git@github.com:sh3r44z/homelab-ansible.git
cd homelab-ansible
```

**2. Update the inventory**

Edit `inventory.ini` with your server's IP and username:
```ini
[homelab]
192.168.0.10

[homelab:vars]
ansible_user=your_username
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**3. Run the playbook**
```bash
ansible-playbook -i inventory.ini site.yml --ask-become-pass
```

Ansible will prompt for your sudo password, then work through each level. The whole run takes about 3 to 5 minutes on a fresh server.

---

## Ports opened by the firewall

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3030 | Grafana |
| 9090 | Prometheus |
| 9093 | Alertmanager |
| 3001 | Uptime Kuma |
| 5678 | n8n |

Everything else is blocked by default.

---

## What I ran into building this

Getting here took about 6 hours and a fair amount of debugging. A broken Docker repo entry was conflicting with apt and breaking even basic package updates. Port 8080 was already allocated by another container which stopped cAdvisor from starting. The DNS on the server was pointing at a Docker container IP that wasn't responding, which was causing slow package downloads and failed hostname resolution. Docker was already installed on the target machine which required rewriting the Docker role to detect this and skip installation gracefully rather than failing on a conflicting repository entry.

None of that is in the playbook anymore because Ansible handles it. But it's worth knowing those problems exist if you're running this against a server that already has services on it.

---

## What I learned

Ansible's agentless model. It only needs SSH, nothing installed on the target. How idempotency works in practice and why it matters for running playbooks safely on live servers. Writing handlers so SSH restarts only once after config changes rather than on every task. Using the `when` conditional to make roles smart enough to skip steps that aren't needed. Troubleshooting real networking and port conflicts on a live home lab.

---

