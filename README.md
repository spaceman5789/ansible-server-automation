# Ansible Server Automation: Nginx + Docker + HTTPS

> One playbook turns a fresh Debian/Ubuntu VM into a near-production host: Docker app deployed from Git, managed by systemd, behind an Nginx reverse proxy with HTTPS and a UFW firewall.

![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?logo=nginx&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-003A70?logo=letsencrypt&logoColor=white)
![Debian](https://img.shields.io/badge/Debian%20%2F%20Ubuntu-A81D33?logo=debian&logoColor=white)

---

## Contents

- [What It Does](#what-it-does)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Variables](#variables)
- [Verify the Deployment](#verify-the-deployment)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [Security Notes](#security-notes)

---

## What It Does

Running `site.yml` against a host will:

1. Install packages: `nginx`, `git`, `docker.io`, `docker-compose`, `ufw`, `openssl`, `certbot`
2. Create an unprivileged `deploy` user in the `docker` group
3. Clone your application repository into `/opt/demo-app`
4. Render a **systemd unit** that runs the app (`docker-compose up -d`) and restarts it on changes
5. Configure **Nginx** as a reverse proxy in front of the app
6. Lock down the host with **UFW**: deny incoming by default, allow `22`, `80`, `443`
7. Enable **HTTPS** — either a self-signed certificate (default) or **Let's Encrypt** via certbot when a domain is set

The playbook is idempotent: re-running it only changes what drifted.

## Architecture

```mermaid
flowchart LR
    U[Client] -->|80 / 443| UFW[UFW firewall]
    UFW --> N[Nginx<br/>TLS termination]
    N -->|proxy_pass :app_port| A[App containers<br/>docker-compose]
    S[systemd unit<br/>demo-app.service] -. manages .-> A
    G[(Git repo)] -. ansible.builtin.git .-> A
```

## Requirements

**Control machine**

- Ansible 2.14+
- `community.general` collection (installed below)

**Target host**

- Debian 11+/Ubuntu 22.04+ with SSH access and sudo
- A domain pointing at the host if you want Let's Encrypt

## Quick Start

```bash
# 1. Describe your server
cp ansible/inventory.example ansible/inventory
$EDITOR ansible/inventory

# 2. Install dependencies
ansible-galaxy collection install -r ansible/requirements.yml

# 3. Run
ansible-playbook -i ansible/inventory ansible/site.yml
```

Example inventory (SSH key auth):

```ini
[web]
app1 ansible_host=203.0.113.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519

[web:vars]
app_repo=https://github.com/spaceman5789/nginx-postgres-monitoring
app_port=8080
# ssl_mode=letsencrypt
# domain_name=example.com
# letsencrypt_email=admin@example.com
```

> `ansible/inventory` is git-ignored — keep real hosts and credentials out of the repo. For passwords use `ansible-vault` or `--ask-pass --ask-become-pass`.

## Variables

Defaults live in `ansible/group_vars/all.yml` and can be overridden per host/group:

| Variable | Default | Description |
|----------|---------|-------------|
| `app_repo` | `https://github.com/example/example-app.git` | Git repository to deploy |
| `app_version` | `main` | Branch, tag or commit |
| `app_name` | `demo-app` | systemd service name |
| `app_dir` | `/opt/demo-app` | Checkout directory |
| `app_port` | `8080` | Port Nginx proxies to |
| `app_exec_start` / `app_exec_stop` | `docker-compose up -d` / `down` | Commands used by the systemd unit |
| `deploy_user` | `deploy` | Owner of the app directory |
| `ssl_mode` | `selfsigned` | `selfsigned` or `letsencrypt` |
| `domain_name` | `example.com` | Server name / certificate CN |
| `letsencrypt_email` | `admin@example.com` | Contact for Let's Encrypt |

## Verify the Deployment

```bash
ansible app1 -i ansible/inventory -b -m shell -a "systemctl status demo-app --no-pager -l"
ansible app1 -i ansible/inventory -b -m shell -a "cd /opt/demo-app && docker-compose ps"
ansible app1 -i ansible/inventory -b -m shell -a "systemctl status nginx --no-pager"
curl -kI https://<host>
```

## Troubleshooting

**The first run looks stuck.** Usually Docker is pulling large images. Pre-pull them and re-run the playbook:

```bash
ansible app1 -i ansible/inventory -b -m shell -a "cd /opt/demo-app && docker-compose pull"
```

**Let's Encrypt fails.** Check that the domain resolves to the host and port 80 is reachable from the internet.

## Project Structure

```text
ansible/
├── ansible.cfg
├── inventory.example          # copy to ./inventory
├── requirements.yml           # community.general
├── group_vars/
│   └── all.yml                # default variables
├── site.yml                   # main playbook
└── templates/
    ├── app.service.j2         # systemd unit
    └── nginx-app.conf.j2      # reverse proxy + TLS
```

## Security Notes

- Default-deny firewall, only SSH/HTTP/HTTPS exposed
- The application runs under a dedicated `deploy` user
- TLS everywhere; Let's Encrypt with automatic HTTP→HTTPS redirect

Ideas for hardening: SSH key-only login and `fail2ban`, rate limiting and security headers in Nginx, splitting the playbook into roles and testing them with Molecule.

---

## Related Projects

- [nginx-postgres-monitoring](https://github.com/spaceman5789/nginx-postgres-monitoring) — the Docker Compose app this playbook deploys by default
- [one-click-infrastructure](https://github.com/spaceman5789/one-click-infrastructure) — Terraform + Ansible roles across two VMs

## Author

**Denis Nazarov** — DevOps Engineer · [GitHub](https://github.com/spaceman5789) · [Portfolio](https://denzo-portfolio.site)
