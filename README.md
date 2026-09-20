# Ansible Linux Lab

A learning repository for disciplined Ansible automation. The example role configures Nginx on Debian-family systems and serves a small page. The project demonstrates inventory as code, reusable role defaults, privilege escalation, idempotent tasks, validated handlers, staged rollouts, and post-deployment health checks.

> **Scope:** This is a production-oriented learning foundation, not a security-hardened production web service. The sample serves HTTP without TLS, and the role takes ownership of the host's default Nginx site on port 80. Use disposable lab hosts until your organization has reviewed the role, operating system support, security controls, and change process.

## Architecture

```text
Kali control node
  └── Ansible inventory and playbooks
       ├── Local lab target: Kali itself (learning only)
       ├── Future lab targets: Debian/Ubuntu servers over SSH
       └── Separate production inventory: explicit opt-in at runtime
```

Kali is supported here only as the current local learning target because it uses the Debian package family. For real managed hosts, prefer a supported Debian or Ubuntu server image. Do not treat a security-testing workstation as a production server.

## Repository layout

```text
ansible.cfg                         Safe defaults and lab inventory path
site.yml                            Validates targets, rolls out in batches, checks health
inventories/lab/hosts.yml           Versioned non-secret lab inventory
inventories/lab/group_vars/         Lab-specific non-secret settings
inventories/production/             Production inventory example
roles/web/defaults/                 Overrideable role defaults
roles/web/tasks/                    Idempotent host configuration
roles/web/handlers/                  Validated Nginx reload handler
roles/web/templates/                 Managed page and Nginx site templates
.ansible-lint                       Production rules profile
.gitignore                          Excludes local inventory and secret files
```

## Prerequisites

- Kali Linux as the Ansible control node for the first exercise
- Ansible installed on Kali; Git for cloning the repository
- `sudo` access on the local learning target
- For future remote hosts: Debian or Ubuntu, SSH access, Python 3, and a sudo-capable automation account

Install the tools on Kali if needed:

```bash
sudo apt update
sudo apt install -y ansible git openssh-client
ansible --version
```

## First run: Kali as a local learning target

The committed lab inventory already targets Kali through a local connection, so no local inventory copy or SSH setup is needed:

```bash
git clone https://github.com/harshit3355/ansible-linux-lab.git
cd ansible-linux-lab
ansible-inventory --graph
ansible webservers -m ansible.builtin.ping
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml --ask-become-pass
```

The first run may ask for your Kali sudo password. Open `http://localhost/` to see the page. Run the playbook again and inspect the `changed` count; a stable second run should report no changes. Check mode is a preview, and some Ansible modules cannot predict every change, so review its output rather than treating it as a guarantee.

The play runs hosts in batches of ten (`serial: 10`) and stops the play if a host fails. It validates the operating-system family and inputs before changing the host. After deployment, it requests the page locally and checks the response content. Health checks are skipped in check mode.

## Add a lab server later

Add servers to the `webservers` group in `inventories/lab/hosts.yml`. Keep host-specific connection details there; keep shared settings in `inventories/lab/group_vars/webservers.yml` and exceptional, non-secret settings in `inventories/lab/host_vars/<inventory-name>.yml`.

For example, extend the hosts section:

```yaml
all:
  children:
    webservers:
      hosts:
        kali_local:
          ansible_connection: local
        web-01:
          ansible_host: 192.168.56.21
          ansible_user: ansible
        web-02:
          ansible_host: 192.168.56.22
          ansible_user: ansible
```

Use stable inventory names such as `web-01`; `ansible_host` can be an address or DNS name. Before adding a host, confirm SSH from Kali and install the automation account's public key on the target. Example:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_lab
ssh-copy-id -i ~/.ssh/ansible_lab.pub ansible@192.168.56.21
ssh -i ~/.ssh/ansible_lab ansible@192.168.56.21
```

Load the key into your SSH agent (or configure a host alias and `IdentityFile` in `~/.ssh/config`) so Ansible can use it:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/ansible_lab
```

Do not put private keys, passwords, tokens, or private key contents in inventory or Git. Use SSH agent/key management for credentials. Avoid committing machine-specific key paths.

Review the inventory and target scope before running:

```bash
ansible-inventory --graph
ansible webservers --list-hosts
ansible web-01 -m ansible.builtin.ping
ansible-playbook site.yml --limit web-01 --check --diff
ansible-playbook site.yml --limit web-01 --ask-become-pass
```

Deploy to one canary first, inspect the result, then run against the full group. `serial: 10` controls batch size; change it deliberately for your rollout policy. Use `--limit` for a specific host or group, and avoid `--limit all` unless that scope is intended.

## Keep environments separate

The default inventory is `inventories/lab/hosts.yml`. Production hosts are not included in it. Start from the example and create a local production inventory:

```bash
cp inventories/production/hosts.example.yml inventories/production/hosts.yml
nano inventories/production/hosts.yml
```

`inventories/production/hosts.yml` is ignored by Git by default. This is a safe default for private infrastructure details; teams may instead version-control sanitized inventories under their own policy. Use the production inventory explicitly and preview a canary before applying:

```bash
ansible-inventory -i inventories/production/hosts.yml --graph
ansible-playbook -i inventories/production/hosts.yml site.yml --limit web-01 --check --diff
ansible-playbook -i inventories/production/hosts.yml site.yml --limit web-01 --ask-become-pass
```

Do not point the lab inventory at production. Keep environment variables and inventories separate, require review for production changes, and use your team's change window and rollback process.

## Secrets

Never store plaintext secrets, SSH private keys, or vault passwords in Git. For Ansible variables that must be versioned, use Ansible Vault and provide the vault password through an approved secret manager or an interactive prompt:

```bash
mkdir -p inventories/production/group_vars/webservers
ansible-vault create inventories/production/group_vars/webservers/vault.yml
ansible-playbook -i inventories/production/hosts.yml site.yml --ask-vault-pass
```

Review encrypted files too: encryption does not protect secrets that were committed elsewhere in plaintext. If a secret is exposed, rotate it.

## Variables and customization

Defaults live in `roles/web/defaults/main.yml`. Override them in an inventory's `group_vars` or `host_vars`, not by editing task files. Example:

```yaml
web_page_title: Internal Status Page
web_page_heading: Staging web node
web_listen_port: 8080
```

The role validates the port and required text before applying changes. The web role currently supports Debian-family hosts and manages Nginx plus Git. It configures the default HTTP site and removes the packaged default-site symlink so there is one clear owner for port 80.

## Quality and change workflow

For each change:

1. Update defaults or inventory data before changing reusable tasks.
2. Run `ansible-inventory --graph` and confirm target membership.
3. Run syntax checks and `ansible-playbook ... --check --diff` against a disposable host or canary.
4. Review the diff and host limit, then apply to a canary.
5. Confirm the health check and inspect the second run for idempotence.
6. Commit only reviewed source, sanitized inventory examples, and encrypted secrets where required.

The repository includes an `ansible-lint` production profile. Install `ansible-lint` in a dedicated Python virtual environment and run it before proposing changes:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install ansible-lint
ansible-lint
```

Pin approved `ansible-core` and `ansible-lint` versions in your team's CI/tooling image as this project grows. CI, approvals, signed artifacts, TLS, firewall policy, monitoring, backups, and operating-system hardening are future project work; this repository does not claim to implement them yet.

## Troubleshooting

- **Ansible encoding error:** in Kali, set `export LANG=C.UTF-8 LC_ALL=C.UTF-8 PYTHONUTF8=1` for the current shell, then retry.
- **Unreachable remote host:** verify the inventory address and test `ssh ansible@<host>` from Kali.
- **Permission denied:** check the SSH user/key and target account's authorized keys.
- **Sudo failure:** confirm the automation account can use sudo; use `--ask-become-pass` when password-based escalation is configured.
- **Nginx config failure:** inspect `sudo nginx -t` on the target and review the latest Ansible task output before retrying.
- **Port 80 already in use:** inspect existing services and site configuration before applying; the role intentionally manages the default Nginx site.

## Publishing checklist

- Confirm `inventories/production/hosts.yml`, local inventory overrides, vault passwords, keys, tokens, and plaintext secrets are not staged.
- Inspect `git diff --cached` before committing.
- Keep host identifiers and addresses within your organization's disclosure policy.
- Publish only the example inventories and documentation that you are comfortable making public.
