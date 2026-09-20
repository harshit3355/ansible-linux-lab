# Ansible Linux Lab

Learn configuration management by using Ansible to prepare a Debian or Ubuntu web server. The playbook installs Nginx and Git, publishes a small lab homepage, and makes sure Nginx is enabled and running.

## Lab layout

Use Kali as the Ansible control node and a separate Debian or Ubuntu VM as the managed node. Keeping the managed node separate makes it easier to practice inventory, SSH, and privilege escalation. Both VMs should be on a VMware network where they can reach each other.

```text
Kali (Ansible control node) -- SSH --> Debian/Ubuntu (managed node)
                                      Nginx serves the lab page
```

## Prerequisites

- Kali Linux with Ansible installed
- A Debian or Ubuntu VM with a reachable IP address
- SSH access to that VM and a user that can run `sudo`
- Network connectivity from Kali to the managed VM

On Kali, install Ansible if needed:

```bash
sudo apt update
sudo apt install ansible openssh-client
ansible --version
```

On the managed VM, find its IP address with `ip addr`. Confirm SSH is enabled and that Kali can connect:

```bash
ssh <username>@<managed-vm-ip>
```

## Configure the inventory

Copy the example inventory and replace the sample host, username, and key path with your VM details:

```bash
cp inventory.ini.example inventory.ini
nano inventory.ini
```

The example uses SSH keys. If you do not have a key yet, create one on Kali and install its public key on the managed VM:

```bash
ssh-keygen -t ed25519
ssh-copy-id <username>@<managed-vm-ip>
```

Keep private keys out of this repository. `inventory.ini` is ignored by Git; commit only `inventory.ini.example` with placeholder values.

## Run the playbook

From this project directory on Kali:

```bash
ansible-inventory --list -i inventory.ini
ansible webservers -m ping
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

Open `http://<managed-vm-ip>/` in a browser, or use:

```bash
curl http://<managed-vm-ip>/
```

Run the playbook again. Ansible should report no changes after the first successful run. That repeatability is called **idempotence**.

## What the files do

- `ansible.cfg` selects the local inventory and enables useful output.
- `inventory.ini.example` shows how to describe the managed host.
- `site.yml` targets the `webservers` group and applies the web role.
- `roles/web/tasks/main.yml` installs packages, publishes the page, and starts Nginx.
- `roles/web/templates/index.html.j2` is the page template rendered on the managed host.

## Practice tasks

1. Change the page title and deploy it again.
2. Add a variable for the page heading in `group_vars/webservers.yml` and use it in the template.
3. Add a handler so Nginx restarts only when its configuration changes.
4. Add a second managed VM to the inventory and deploy to both.

## Troubleshooting

- **Unreachable host:** check the IP, VMware network mode, and `ssh <username>@<managed-vm-ip>` from Kali.
- **Permission denied:** confirm the SSH username and key; test with `ssh -i <key-path> <username>@<managed-vm-ip>`.
- **Sudo prompt/failure:** ensure the SSH user has sudo rights. The playbook uses `become: true` and may prompt for the sudo password.
- **Nginx page not reachable:** check `sudo systemctl status nginx` on the managed VM and verify the VM firewall/network allows HTTP.

## Before publishing

Review the files and confirm that no private key, password, or real secret is included. Commit the example inventory, never your local `inventory.ini`.
