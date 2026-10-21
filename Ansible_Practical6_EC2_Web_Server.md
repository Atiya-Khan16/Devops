# Ansible Practical 6: AWS EC2 Web Server Automation

## 1. Aim

Automate an AWS EC2 Ubuntu server using Ansible: install Nginx, ensure the service is running, and deploy a custom HTML page.

## 2. Architecture

Windows laptop → Ubuntu on WSL (Ansible control node) → SSH → AWS EC2 Ubuntu (managed node) → Nginx → Web page

## 3. Requirements

- Windows with Ubuntu WSL (or another Linux control node)
- Ansible installed on the control node
- A running Ubuntu EC2 instance
- The matching EC2 `.pem` private key
- EC2 public IPv4 address
- EC2 security-group inbound rules for SSH (22) from **My IP** and HTTP (80) from `0.0.0.0/0`
- Internet access from the EC2 instance to install packages

> Replace `YOUR_EC2_PUBLIC_IP` with the instance's current public IPv4 address. Replace the example key path and filename with your actual values. Never share or commit your private key.

## 4. Install Ansible (Ubuntu WSL)

Open the **Ubuntu** terminal in Windows, then run:

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

If `ansible --version` prints version information, Ansible is installed.

## 5. Copy and protect the EC2 key

Example assumes the key is in your Windows Downloads folder and the Windows username is `lenovo`. Change the path if yours differs.

```bash
mkdir -p ~/.ssh
cp /mnt/c/Users/lenovo/Downloads/ansible-key.pem ~/.ssh/
chmod 400 ~/.ssh/ansible-key.pem
ls -l ~/.ssh/ansible-key.pem
```

The `chmod 400` command restricts access to the key. If the key is already stored in another location, copy it from that location instead.

## 6. Test SSH to EC2

```bash
ssh -i ~/.ssh/ansible-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

- On the first connection, verify the host fingerprint if possible and type `yes` to continue.
- A successful login shows the remote Ubuntu shell.
- Type `exit` to return to the WSL control node.

If SSH fails, check the instance state, public IP, key pair, username (`ubuntu` for standard Ubuntu AMIs), and security-group port 22 rule.

## 7. Create the project directory

Run on the **WSL control node**, not inside the EC2 SSH session:

```bash
mkdir -p ~/ansible-practical
cd ~/ansible-practical
pwd
```

## 8. Create `inventory.ini`

Create the inventory file:

```bash
nano inventory.ini
```

Paste this, replacing the public IP and key path if needed:

```ini
[webserver]
aws-server ansible_host=YOUR_EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=/home/ YOUR_WSL_USERNAME/.ssh/ansible-key.pem
```

**Important:** Remove the space after `/home/` and replace `YOUR_WSL_USERNAME` with your actual Linux username. To find it, run `whoami`. For example, if `whoami` prints `lenovo`, the path is `/home/lenovo/.ssh/ansible-key.pem`.

A safer alternative is to use the home shortcut in the inventory:

```ini
[webserver]
aws-server ansible_host=YOUR_EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/ansible-key.pem
```

Save in nano: `Ctrl+O`, press `Enter`, then `Ctrl+X`.

Check the file:

```bash
cat inventory.ini
```

## 9. Test Ansible connectivity

```bash
ansible -i inventory.ini webserver -m ping
```

Expected result includes `SUCCESS` and `"ping": "pong"`.

An interpreter-discovery warning can appear and is often harmless if the ping succeeds. If the result is `UNREACHABLE`, first test the SSH command directly and check the inventory IP, username, and key path.

## 10. Create the HTML page

```bash
nano index.html
```

Paste and save:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Ansible Practical</title>
</head>
<body>
  <h1>Hello from Ansible!</h1>
  <p>This webpage was deployed automatically to AWS EC2.</p>
</body>
</html>
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

## 11. Create the Ansible playbook `nginx.yml`

```bash
nano nginx.yml
```

Paste this complete playbook:

```yaml
---
- name: Deploy Web Application
  hosts: webserver
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is started and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy HTML page
      ansible.builtin.copy:
        src: index.html
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: '0644'
```

Save the file, then validate the YAML/playbook syntax:

```bash
ansible-playbook -i inventory.ini nginx.yml --syntax-check
```

Expected: `playbook: nginx.yml` with no syntax errors.

## 12. Run the playbook

From `~/ansible-practical` on WSL:

```bash
ansible-playbook -i inventory.ini nginx.yml
```

The playbook installs Nginx, starts and enables it, and copies `index.html` to `/var/www/html/index.html`. `become: true` uses privilege escalation on EC2; enter the SSH user's sudo password only if prompted.

A successful recap should show `unreachable=0` and `failed=0`. The `changed` count can differ on later runs because Ansible is idempotent.

## 13. Verify the result

Check Nginx service status:

```bash
ansible -i inventory.ini webserver -a "systemctl status nginx --no-pager"
```

Check the HTTP response locally on EC2:

```bash
ansible -i inventory.ini webserver -a "curl -I http://localhost"
```

Expected response includes `HTTP/1.1 200 OK`.

Check the deployed HTML:

```bash
ansible -i inventory.ini webserver -a "cat /var/www/html/index.html"
```

Then open this address in your browser, replacing the placeholder:

```text
http://YOUR_EC2_PUBLIC_IP
```

The page should show **Hello from Ansible!** and the message that the webpage was deployed automatically.

## 14. AWS security-group setup

In AWS Console:

1. Open **EC2 → Instances** and select your instance.
2. Open its **Security** tab and select the attached security group.
3. Choose **Edit inbound rules**.
4. Add or verify these rules:

| Type | Protocol | Port | Source |
|---|---|---:|---|
| SSH | TCP | 22 | My IP |
| HTTP | TCP | 80 | `0.0.0.0/0` |

5. Save the rules.

Keep SSH restricted to **My IP** where practical. HTTP is public in this example because the webpage is intended to be reachable from a browser.

## 15. Troubleshooting

### A. `Permission denied (publickey)`

- Confirm you are using the key pair attached to this EC2 instance.
- Confirm the login user is `ubuntu` for a standard Ubuntu image.
- Check `chmod 400 ~/.ssh/ansible-key.pem`.
- Test direct SSH before testing Ansible.

```bash
ssh -i ~/.ssh/ansible-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

### B. `UNREACHABLE` or SSH timeout

- Confirm EC2 is running and has a reachable public IPv4 address.
- Check inbound TCP 22 allows your current public IP.
- Check the IP and key path in `inventory.ini`.
- Ensure the network you are using allows outbound SSH.

### C. `Could not resolve hostname` or inventory parsing error

Inspect the inventory:

```bash
cat -n inventory.ini
ansible-inventory -i inventory.ini --graph
```

Make sure the host line is under `[webserver]`, the IP is correct, and the key path contains no accidental spaces.

### D. `Permission denied` while copying the key

Copy the key from the correct Windows path. In WSL, Windows `C:\Users\...` is commonly mounted under `/mnt/c/Users/...`. Do not paste the Windows path directly into Linux commands.

### E. Browser shows `ERR_CONNECTION_REFUSED`, but local curl returns 200

- Confirm the EC2 security group allows inbound TCP port 80 from `0.0.0.0/0`.
- Confirm you are using the current EC2 public IP.
- Check Nginx is running:

```bash
ansible -i inventory.ini webserver -a "systemctl status nginx --no-pager"
```

### F. Nginx is not running

Run the playbook again, then inspect the service:

```bash
ansible-playbook -i inventory.ini nginx.yml
ansible -i inventory.ini webserver -a "systemctl status nginx --no-pager"
```

### G. Playbook syntax error

Check indentation. YAML uses spaces, not tabs. Validate with:

```bash
ansible-playbook -i inventory.ini nginx.yml --syntax-check
```

### H. `apt` lock or package installation failure

Wait for any automatic Ubuntu package updates to finish and retry. Confirm the EC2 instance has internet connectivity and DNS resolution.

## 16. Important commands quick reference

| Purpose | Command |
|---|---|
| Check Ansible version | `ansible --version` |
| Check current WSL username | `whoami` |
| Test SSH | `ssh -i ~/.ssh/ansible-key.pem ubuntu@YOUR_EC2_PUBLIC_IP` |
| Test Ansible connection | `ansible -i inventory.ini webserver -m ping` |
| List inventory hosts | `ansible-inventory -i inventory.ini --graph` |
| Validate playbook | `ansible-playbook -i inventory.ini nginx.yml --syntax-check` |
| Run playbook | `ansible-playbook -i inventory.ini nginx.yml` |
| Check Nginx status | `ansible -i inventory.ini webserver -a "systemctl status nginx --no-pager"` |
| Test local HTTP | `ansible -i inventory.ini webserver -a "curl -I http://localhost"` |
| Display deployed HTML | `ansible -i inventory.ini webserver -a "cat /var/www/html/index.html"` |

## 17. Cleanup (optional)

To avoid ongoing AWS charges, stop or terminate the EC2 instance when you no longer need it. Termination is irreversible for the instance's local data, so save anything needed first. Check whether any EBS volumes or other billable resources remain.

## 18. Result

Ansible was used from Ubuntu WSL to connect to an AWS EC2 Ubuntu server over SSH, install and start Nginx, and deploy a custom HTML page. The page is accessible at `http://YOUR_EC2_PUBLIC_IP` when the EC2 security group allows inbound HTTP traffic.
