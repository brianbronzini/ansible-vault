# HashiCorp Vault Deployment

Two-node HashiCorp Vault setup, fully automated with Ansible. Primary Vault holds the secrets that downstream services consume. A second smaller Vault holds the transit key that auto-unseals the primary on every restart, so the operator only has to manually unseal one node.

Vault audit logs ship to a Wazuh SIEM, so every secret access is visible to the SOC stack.

## Project Purpose

I built this for two reasons. First, my own infrastructure had accumulated secrets in plain `.env` and `config.ini` files scattered across half a dozen hosts. I wanted them in one place, with an audit trail, and managed the same way as everything else I run. Second, I wanted a hands-on project that exercised the patterns I read about for production secrets management: auto-unseal, least-privilege auth, audit forwarding to a SIEM, transport security via a real cert.

The pattern here mirrors what a small DevOps team would use as a starting point before going to a three-node HA cluster with cloud KMS unseal. If any of this is useful to you, fork it and change what you need to. The roles are intentionally small and standalone, so it should be straightforward to lift `vault_install`, `vault_unsealer`, or `vault_primary` into a different deployment without taking the rest. The Wazuh agent role assumes you already have a manager somewhere; if you don't, drop that role from `site.yml` and the rest still converges. Questions, suggestions, and pull requests are welcome! :)

## Project Structure

```
inventory/             Ansible inventory (hosts, groups)
inventory/group_vars/  shared variables per group
inventory/host_vars/   per-host overrides
roles/                 reusable Ansible roles
playbooks/             orchestration playbooks
files/                 static assets copied to hosts
```

Each role has a short comment at the top of `tasks/main.yml` describing what it does.

## Results

The full deployment converges end to end on `ansible-playbook playbooks/site.yml` with zero changes on re-run.

The playbook outcome provides:

- Two Vault nodes, primary auto-unsealed by the unsealer via the Transit engine.
- KV v2 mounted at `kv/`, an `ansible-controller` AppRole bound to a specific controller IP with least-privilege policy.
- File audit on both nodes, Wazuh agents tailing to the manager, custom rules (IDs 100200 to 100207) firing on root token use, policy changes, auth method changes, seal events, and KV access.
- An end-to-end playbook (`playbooks/sync-discord-webhook.yml`) that pulls a secret from Vault via AppRole and templates it onto a downstream consumer.
- Tailscale serve fronts the primary API with a valid Let's Encrypt cert, so the Vault UI is reachable at `https://<hostname>.<tailnet>.ts.net/` from any user-tagged device on the tailnet. The LAN listener stays on 8200 so the Ansible controller keeps working.

## Quick start

Run from your Ansible controller as the user that owns the SSH key referenced in `ansible.cfg`.

```
ansible-playbook playbooks/site.yml
```

This converges everything: baseline hardening, Vault install, unsealer bootstrap, primary bootstrap with transit auto-unseal, KV and audit enablement, Wazuh enrollment, and AppRole setup. Idempotent. Re-running is a no-op.

To prove the AppRole path works end to end:

```
ansible-playbook playbooks/sync-discord-webhook.yml
```

This reads `kv/ansible/jellyseerr` from Vault and applies the Discord webhook to the downstream consumer's settings, restarting the container only if the value changed.

## Notes about this specific deployment

The inventory, paths, and IPs in this repo reflect the homelab it actually runs on. SSH key path, controller user, and LAN CIDR are wired to my setup. If you fork this as a starting point, expect to swap those values in `ansible.cfg`, `inventory/group_vars/all.yml`, and the role defaults.

The Wazuh manager, the Jellyseerr consumer in the sync playbook, and the Proxmox host pattern are all part of my existing infrastructure. The roles themselves stand alone and can be picked up without those pieces.
