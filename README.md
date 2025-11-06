# Home Lab Backup Services

Ansible-based deployment for home-lab backup and lightweight multimedia dashboards.

This repository provides Ansible roles and Jinja2 Docker Compose templates to deploy:

- Restic-based system file backups (containerized)
- Immich multimedia backup services (server, ML worker, Redis, Postgres)
- A small Homepage dashboard for quick access to services

This project is a focused subset of the larger Multimedia Home Lab projects and depends on two companion repositories which provide the broader home-lab foundations and multimedia stack:

- Home Lab base setup: https://github.com/cyph3ramfm/home_lab_setup
- Multimedia Suite (reference): https://github.com/cyph3ramfm/multimedia_suite

Use those repositories to prepare target hosts (Docker, networks, SMB mounts, system users, firewall rules) before deploying this repo's services.

## Features

- System file backups using Restic (containerized Compose services: backup, prune, check)
- Multimedia backup stack using Immich (server, machine-learning worker, Redis, PostgreSQL)
- Homepage dashboard (homepage + services snippets) for quick access and status
- Templates rendered via Ansible into a temporary workspace and deployed using Docker Compose v2 through the `community.docker` collection

## Repo layout

Key files and folders:

- `deploy_home_lab_backup_services_playbook.yml` — top-level playbook that wires roles and loads `group_vars/*.yml`
- `group_vars/main.yml` — main configuration and feature flags
- `group_vars/vault.yml.template` — vault template (secrets) — DO NOT COMMIT real `vault.yml`
- `roles/` — role directories containing tasks and Jinja2 templates
  - `roles/system_files_backup/` — restic role and template (`restic.yml.j2`)
  - `roles/multimedia_backup/` — immich role and templates (`immich.yml.j2`, `hwaccel.transcoding.yml.j2`)
  - `roles/dashboards/` — homepage role and templates (`homepage.yml.j2`, `services.yaml.j2`)

Roles follow a common pattern:

1. Pre-checks (e.g., Docker networks or containers that must exist)
2. Render templates into a temporary directory (usually `/tmp/<role>`)
3. Deploy using `community.docker.docker_compose_v2` against the rendered Compose files

When editing templates, preserve this flow and the pre-check patterns.

## Prerequisites

Before running the playbook, ensure the target hosts and environment are prepared. In particular:

- Docker and Docker Compose v2 installed on target hosts
- Required Docker networks created (`home_lab`, `proxy` are commonly used)
- If using SMB/CIFS for storage, make sure the NAS shares are exported and reachable from the target host
- The two companion repositories (see links above) should be used to prepare the base host and optional multimedia services

## Quick start

Clone this repository and prepare vault secrets:

```bash
git clone <repository-url>
cd home_lab_backup_services
```

Create an Ansible vault password file (one-time):

```bash
# Create a vault password file (protect this file)
echo "your-secure-password" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
```

Create the encrypted vault file from the template and edit it:

```bash
cp group_vars/vault.yml.template group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml --vault-password-file ~/.vault_pass.txt
ansible-vault edit group_vars/vault.yml --vault-password-file ~/.vault_pass.txt
```

Edit general configuration in `group_vars/main.yml` (mount points, domain prefixes, enable/disable features). Update `inventory/hosts` to point at your target host(s).

Deploy the playbook (sudo required on remote host):

```bash
ansible-playbook -i inventory/hosts deploy_home_lab_backup_services_playbook.yml \
  --vault-password-file ~/.vault_pass.txt --ask-become-pass
```

For a dry-run / preview:

```bash
ansible-playbook -i inventory/hosts deploy_home_lab_backup_services_playbook.yml \
  --vault-password-file ~/.vault_pass.txt --ask-become-pass --check
```

If you prefer to enter the vault password interactively use `--ask-vault-pass`.

## Managing secrets (ansible-vault)

This project stores secrets in `group_vars/vault.yml` (encrypted with ansible-vault). `group_vars/vault.yml.template` lists required variables and example formats.

Important secrets include (non-exhaustive):

- `vault_device_name` — unique name for this deployment
- `vault_domain` — base domain for service hostnames
- `vault_restic_password` — restic repository password
- `vault_immich_db_*` — Immich DB credentials

Do NOT commit `group_vars/vault.yml` to version control. Keep `vault.yml.template` in the repo (it should contain anonymized placeholders).

## Troubleshooting

SMB/CIFS (NAS) mount failures

- Error: Failed to mount SMB/CIFS share
  - Check the NAS is reachable: `ping <nas_server>`
  - List SMB shares: `smbclient -L //<nas_server>`
  - Ensure `cifs-utils` is installed on the host (`apt install cifs-utils`)
  - Example `/etc/fstab` entry (use credentials file):

  ```
  //nas_server/multimedia /mnt/multimedia cifs credentials=/root/.smbcredentials,iocharset=utf8,vers=3.0 0 0
  ```

Docker network issues

- Error: network `<name>` not found
  - Create networks manually if needed:

  ```bash
  docker network create home_lab
  docker network create proxy
  ```

Container permission errors

- If you see permission errors connecting to the Docker socket, add the deploy user to the `docker` group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Vault access errors

- ERROR! Attempting to decrypt but no vault secrets found
  - Ensure your vault password file exists and is readable
  - Confirm `group_vars/vault.yml` is encrypted (starts with `$ANSIBLE_VAULT`)

Redeployment issues

- The Ansible Docker Compose deployment may not replace running containers with changed images. To force a redeploy, stop/remove affected containers and re-run the playbook.

Debug mode

- Set `debug_mode: true` in `group_vars/main.yml` to preview rendered templates and see extra debug output from tasks. Roles render templates into `/tmp/<role>` during deployment; this behavior helps inspect the actual Compose files before they are used.

## Role mapping

- `roles/system_files_backup` — renders `restic.yml.j2` and deploys restic backup services via Compose
- `roles/multimedia_backup` — renders `immich.yml.j2` (and optional hwaccel templates) and deploys Immich stack
- `roles/dashboards` — renders homepage templates and deploys a small dashboard

Templates are Jinja2 files under each role's `templates/` directory. When editing templates, ensure referenced variables exist in `group_vars/main.yml` or `group_vars/vault.yml`.

## Security best practices

- Never commit `group_vars/vault.yml` to git — keep only `vault.yml.template` in the repo.
- Store your vault password in a protected file (`~/.vault_pass.txt` with mode `600`) and reference it with `--vault-password-file` for automation.
- Use strong, unique passwords for all service accounts and change default credentials after deployment.
- Prefer HTTPS via Traefik (the templates add Traefik labels). Ensure your Traefik instance and certificates are configured by the base `home_lab_setup` repository.

## Contributing

Contributions are welcome. Suggested workflow:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Keep changes small and focused; follow the existing role/template patterns
4. Update `group_vars/vault.yml.template` if adding new secret variables
5. Submit a pull request with a clear description of changes

## References

- Home Lab base setup: https://github.com/cyph3ramfm/home_lab_setup
- Multimedia Suite (reference): https://github.com/cyph3ramfm/multimedia_suite
