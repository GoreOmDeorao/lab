# Ansible — Practical Commands and Inventories

Commands, inventory examples, and lab procedures from Chef-Puppet-Ansible slides 38–53.

## 1. Install Ansible on Ubuntu

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

If Ansible reports a UTF-8 locale error, add these lines to `~/.bashrc`:

```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

Reload and verify:

```bash
source ~/.bashrc
ansible --version
```

## 2. Create a local inventory and test Ansible

```bash
mkdir -p ~/my_ansible_playbooks
cd ~/my_ansible_playbooks
```

Create `hosts`:

```ini
[local]
localhost ansible_connection=local
```

Test connectivity:

```bash
ansible all -i hosts -m ping
```

## 3. Run the source-deck first playbook

The slide instructs you to download `first.yml` from the linked demo repository. The slide deck does **not** include the body of `first.yml`.

Source link:

- https://github.com/newdelthis/demo/blob/main/first.yml

After placing `first.yml` in `~/my_ansible_playbooks`, run:

```bash
cd ~/my_ansible_playbooks
ansible-playbook --syntax-check first.yml
ansible-playbook -i hosts --check first.yml
ansible-playbook -i hosts first.yml -K
```

`-K` prompts for the privilege-escalation password.

If Apache or NGINX already occupies port 80 and prevents the lab service from starting:

```bash
sudo ss -tulpn | grep ':80'
sudo systemctl stop apache2
sudo systemctl disable apache2
# Or stop NGINX if that is the service using port 80:
sudo systemctl stop nginx
sudo systemctl disable nginx
```

## 4. Set a default inventory file

Option A — specify the inventory with `-i`:

```bash
ansible-playbook -i hosts first.yml
```

Option B — create `ansible.cfg` in the same project directory:

```ini
[defaults]
inventory = hosts
```

Then run:

```bash
ansible-playbook first.yml
```

## 5. Prepare a remote Linux client

Run these commands on the **client** machine:

```bash
sudo apt-get update
sudo apt-get install -y openssh-server
sudo systemctl start ssh
sudo systemctl enable ssh
ip addr
```

Record the client's IP address.

On the **Ansible control node**, add the client to `hosts`:

```ini
[labclients]
<CLIENT_IP> ansible_user=<CLIENT_USERNAME>
```

Verify inventory parsing:

```bash
ansible-inventory -i hosts --list
```

## 6. Configure SSH key authentication

On the Ansible control node:

```bash
ssh-keygen
# Press Enter through the prompts for the lab key.
ssh-copy-id <CLIENT_USERNAME>@<CLIENT_IP>
ssh <CLIENT_USERNAME>@<CLIENT_IP>
hostname
exit
```

On the client, inspect key installation and permissions:

```bash
cat ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

The slide's SSH server configuration steps are:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure these settings are present if your lab requires them:

```text
PubkeyAuthentication yes
PasswordAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

From the control node, verify login:

```bash
ssh <CLIENT_USERNAME>@<CLIENT_IP>
hostname
exit
```

If your key file is named `id_rsa` and you need to specify it:

```bash
ssh -i ~/.ssh/id_rsa <CLIENT_USERNAME>@<CLIENT_IP>
```

Do not copy the private key to the client. Only the public key belongs in the client's `authorized_keys` file. Enabling `PasswordAuthentication yes` is shown because the slide includes it, but key-based login does not require password authentication; avoid enabling password login unless your lab requires it.

## 7. Run ad-hoc commands on clients

```bash
# Test all hosts
ansible all -i hosts -m ping

# Execute commands on all hosts
ansible all -i hosts -a "uptime"
ansible all -i hosts -a "ip a"
ansible all -i hosts -a "ls -l /home/<CLIENT_USERNAME>"

# List installed Debian packages
ansible all -i hosts -a "dpkg -l"
```

## 8. Copy a file from control node to client — source file reference

The slide says to create and run `copy.yml`, but does not show its YAML contents. The control-node steps shown are:

```bash
echo "I am testing ansible" > /home/<CONTROL_NODE_USERNAME>/testfile.txt
nano copy.yml
ansible-playbook -i hosts copy.yml
```

The slides then demonstrate adding the remote user to the inventory, for example:

```ini
[labclients]
<CLIENT_IP> ansible_user=<CLIENT_USERNAME>
```

Run the playbook again and verify on the client:

```bash
ls -l /tmp/testfile.txt
cat /tmp/testfile.txt
```

Do not copy the source slide's example `ansible_become_pass=atul` into a real inventory. It stores a plaintext sudo password in a file. Use `-K` to prompt for the password instead.

## 9. Change file permissions — source file reference

On the client:

```bash
mkdir -p ~/test
cd ~/test
touch one two three
ls -l
```

On the control node, create the `filepermission.yml` playbook as referenced by the slides, then run:

```bash
ansible-playbook -i hosts filepermission.yml
```

Verify on the client:

```bash
ls -l ~/test
```

The slide deck does not include the YAML body of `filepermission.yml`, so it is not recreated here.

## 10. Cron job verification and cleanup

The source slide refers to a `cronjob.yml` playbook but does not include its YAML body. After running the playbook, use these commands on the client to inspect or remove the scheduled entry:

```bash
crontab -u <CLIENT_USERNAME> -l
sudo crontab -u <CLIENT_USERNAME> -e
# Remove the lab-created cron entry in the editor, then save and exit.
sudo rm -f newfile
ls -l newfile
```

Wait for the scheduled interval and verify that `newfile` is not recreated after removing the cron entry.

## 11. Other referenced Ansible playbooks

The slides refer to these playbooks but do not include their YAML contents:

| File | Task described in the slides | Source slide |
|---|---|---:|
| `httpsetup.yml` | Install and start Apache on the client | 50 |
| `shellscript.yml` | Run a shell script on the client | 51 |
| `cronjob.yml` | Schedule a cron job for user `atul` | 52 |
| `adduser.yml` | Create a new user on the client | 53 |

The slide procedures say to run them with `ansible-playbook <file>.yml` and verify the result on the client. Their actual YAML task definitions are not present in the uploaded source material, so this guide does not invent them.

Source reference: Chef-Puppet-Ansible slides 38–53.
