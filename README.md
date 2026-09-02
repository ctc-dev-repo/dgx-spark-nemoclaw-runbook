# DGX Spark Multi-Model AI Development — Ansible Automation (v2)

Automates deployment of the simplified 2-device architecture:
- **DGX Spark** — inference server (NemoClaw + cold model services)
- **Jetson Orin Nano** — all orchestration (triage model + MCP server +
  SSH tunnel gateway + Prometheus + Grafana)

## Setup

```bash
# 1. Install Ansible and collections
brew install ansible
cd ansible/
ansible-galaxy collection install -r requirements.yml

# 2. Create inventory
cp inventory.ini.example inventory.ini
# Edit with your actual IPs and usernames

# 3. Create and encrypt secrets
cp group_vars/vault.yml.example group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml
ansible-vault edit group_vars/vault.yml

# 4. Gitignore secrets
echo "group_vars/vault.yml" >> .gitignore
echo "inventory.ini" >> .gitignore
```

## Preflight Check

```bash
ansible-playbook site.yml --syntax-check --ask-vault-pass
ansible all -m ping --ask-vault-pass
```

## Deploy

```bash
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

## Deploy Individual Hosts

```bash
# DGX Spark only
ansible-playbook site.yml --limit dgx_spark --ask-vault-pass --ask-become-pass

# Orin Nano only (all orchestration roles)
ansible-playbook site.yml --limit orin_nano --ask-vault-pass --ask-become-pass
```

## Verify Deployment

```bash
ansible-playbook playbooks/verify.yml --ask-vault-pass
```

## Architecture Notes

- Tunnel endpoints bind to **127.0.0.1** on the Orin Nano — cold model APIs
  are never exposed on the LAN. Only the MCP server process can reach them.
- The tunnel_gateway role generates its own ed25519 key and automatically
  authorizes it on the DGX Spark during deployment.
- NemoClaw installer URL must be validated against current NVIDIA docs before
  running: https://build.nvidia.com/spark/nemoclaw/instructions
- Triage model download (~2.5 GB) runs once; subsequent runs are idempotent.
- Role execution order on Orin Nano is deliberate:
  triage_model → mcp_server → tunnel_gateway → monitoring
