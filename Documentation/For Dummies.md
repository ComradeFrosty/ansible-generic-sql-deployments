# Ansible Windows Lab — Setup Guide

> Get from zero to a working Ansible → Windows Server connection in one sitting.
> No prior Ansible experience needed.

---

## What You Are Building

```
Your Windows Machine
├── WSL2 (Ubuntu)          ← Ansible runs here — the "brain"
└── VirtualBox Windows VM  ← Ansible targets this — the "hands"

WSL2 talks to the VM over HTTPS port 5986 (WinRM) — fully encrypted.
```

**Why WSL2?** Ansible does not run natively on Windows. WSL2 gives you a real Linux environment inside Windows where Ansible lives. It then reaches out to Windows targets over the network — exactly how it works in enterprise.

---

## Prerequisites

| What | Where to get it |
|---|---|
| WSL2 with Ubuntu | `wsl --install` in PowerShell (Admin) then restart |
| VirtualBox | [virtualbox.org](https://www.virtualbox.org) |
| Windows Server 2022 ISO | [Microsoft Evaluation Centre](https://www.microsoft.com/en-us/evalcenter/) — free 180-day eval |
| GitHub account | [github.com](https://github.com) |

---

## Step 1 — Install Ansible in WSL2

Open your Ubuntu (WSL2) terminal and run:

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
pip3 install pywinrm --break-system-packages
```

Verify:

```bash
ansible --version
python3 -c "import winrm; print('pywinrm OK')"
```

Both should return without errors.

---

## Step 2 — Set Up the Windows Server VM

1. Create a new VM in VirtualBox — Windows Server 2022, 2GB RAM, 40GB disk
2. **Network adapter → Bridged Adapter → your Wi-Fi adapter**
   - This gives the VM a real IP on your network so WSL2 can reach it
3. Install Windows Server 2022 from the ISO
4. Boot into the VM and open **PowerShell as Administrator**

**Configure WinRM (HTTPS only):**

```powershell
# Step A — Initial WinRM setup
winrm quickconfig -y

# Step B — Allow Basic authentication (username/password inside HTTPS)
winrm set winrm/config/service/Auth '@{Basic="true"}'

# Step C — Create SSL certificate (valid for 5 years)
$cert = New-SelfSignedCertificate `
  -DnsName $env:COMPUTERNAME `
  -CertStoreLocation "Cert:\LocalMachine\My" `
  -KeyUsage KeyEncipherment, DigitalSignature `
  -NotAfter (Get-Date).AddYears(5)

# Note your thumbprint — needed next
$cert.Thumbprint

# Step D — Create HTTPS listener using the certificate
winrm create winrm/config/Listener?Address=*+Transport=HTTPS `
  @{CertificateThumbprint="$($cert.Thumbprint)"}

# Step E — Open port 5986 in Windows Firewall
New-NetFirewallRule `
  -DisplayName "WinRM HTTPS" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 5986 `
  -Action Allow

# Step F — Export certificate so Ansible can trust it
Export-Certificate `
  -Cert "Cert:\LocalMachine\My\$($cert.Thumbprint)" `
  -FilePath "C:\Users\Administrator\Desktop\WinRmCert.cer"
```

**Set a static IP so it never changes:**

```powershell
# Find your interface name first
Get-NetIPConfiguration

# Set static IP (use an IP outside your router's DHCP range)
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress "<CHOSEN-STATIC-IP>" `
  -PrefixLength 24 `
  -DefaultGateway "<YOUR-ROUTER-IP>"
```

---

## Step 3 — Get the Certificate into WSL2

**On the VM** — serve the cert file temporarily:

```powershell
cd C:\Users\Administrator\Desktop
python -m http.server 8080
```

**In WSL2** — download and convert it:

```bash
# Download
wget http://<VM-IP>:8080/WinRmCert.cer -O ~/.ansible/WinRmCert.cer

# Convert from Windows binary (.cer) to Linux text format (.pem)
openssl x509 -inform DER \
  -in ~/.ansible/WinRmCert.cer \
  -out ~/.ansible/WinRmCert.pem

# Lock permissions — Ansible requires this
chmod 600 ~/.ansible/WinRmCert.pem

# Verify conversion worked — you should see certificate details
openssl x509 -in ~/.ansible/WinRmCert.pem -text -noout
```

Stop the Python server on the VM (`Ctrl+C`).

---

## Step 4 — Tell WSL2 How to Find the VM by Name

The certificate was issued to the VM's hostname, not its IP. Add the mapping:

```bash
echo "<VM-STATIC-IP>  <VM-HOSTNAME>" | sudo tee -a /etc/hosts
```

Find the VM hostname by running `hostname` in PowerShell on the VM. It looks like `WIN-XXXXXXXXX`.

---

## Step 5 — Create the Ansible Project

```bash
# Create project in WSL2 native filesystem
mkdir -p ~/ansible-generic-sql-deployments
cd ~/ansible-generic-sql-deployments

# Create folder structure
mkdir -p inventory group_vars \
  roles/generic-package-installer/{tasks,defaults} \
  roles/generic-script-runner/{tasks,defaults} \
  roles/generic-job-executor/{tasks,defaults}
```

**`ansible.cfg`** — tells Ansible where everything lives:

```ini
[defaults]
inventory = ./inventory/hosts.ini
host_key_checking = False
```

**`inventory/hosts.ini`** — your server list (gitignored — never pushed)

**`inventory/hosts.ini.template`** — safe placeholder version (committed to GitHub):

```ini
[dev_servers]
win_server ansible_host=<YOUR-VM-HOSTNAME>

[dev_servers:vars]
ansible_user=<ADMIN-USERNAME>
ansible_password=<YOUR-PASSWORD>
ansible_port=5986
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_scheme=https
ansible_winrm_ca_trust_path=/home/<YOUR-WSL2-USER>/.ansible/WinRmCert.pem
ansible_winrm_server_cert_validation=validate
```

**`.gitignore`** — credentials and certs never go to GitHub:

```gitignore
inventory/hosts.ini
*.pem
*.cer
*.pfx
*.retry
.vscode/
```

---

## Step 6 — Test the Connection

```bash
cd ~/ansible-generic-sql-deployments
ansible dev_servers -m win_ping
```

Expected output:

```
win_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you see `pong` — the full stack is working. Ansible in WSL2 connected to the Windows VM over encrypted HTTPS, authenticated, and got a response.

---

## Step 7 — Push to GitHub

```bash
git init
git remote add origin https://github.com/<USERNAME>/<REPO-NAME>.git
git add .
git status   # Verify hosts.ini is NOT listed
git commit -m "initial project structure"
git branch -M main
git pull origin main --rebase   # Required if GitHub created a README on init
git push -u origin main
```

> **Password prompt?** GitHub no longer accepts account passwords for Git operations.
> Use a **Personal Access Token (PAT)** instead.
> Generate one at: `GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)`
> Scope needed: `repo` only.

**Save the PAT so you are not prompted every time:**

```bash
git config --global credential.helper store
# Push once with your PAT — it is saved to ~/.git-credentials automatically
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Connection timeout` | VM not reachable on port 5986 | Run `nc -zv <VM-HOSTNAME> 5986` in WSL2. If it fails, check firewall rule on VM. |
| `certificate verify failed: IP address mismatch` | Inventory uses IP but cert was issued to hostname | Use the hostname in inventory, map it in `/etc/hosts` |
| `Authentication failed` | Basic auth not enabled on WinRM | Run `winrm set winrm/config/service/Auth '@{Basic="true"}'` on VM |
| `world writable directory` warning | Running from `/mnt/c/` path | Move project to `~/` inside WSL2 and run from there |
| `Invalid username or token` on push | Using GitHub account password | Use a PAT — generate at GitHub Settings → Developer Settings |
| `failed to push — tip of branch is behind` | Remote has commits local does not | Run `git pull origin main --rebase` then push again |
| Ping fails but port is open | Windows Firewall blocks ICMP by default | Normal — use `nc -zv` not `ping` to test Windows server reachability |

---

## Key Concepts — One Line Each

| Concept | What it means |
|---|---|
| WSL2 | Real Linux kernel inside Windows — where Ansible runs |
| WinRM | Windows Remote Management — Ansible's protocol to talk to Windows (equivalent of SSH) |
| Port 5986 | HTTPS WinRM — always use this, never 5985 (HTTP) |
| Self-signed cert | Provides full encryption — manually trusted instead of CA-trusted |
| `.cer` | Windows certificate format (binary) |
| `.pem` | Linux certificate format (base64 text) — what Ansible reads |
| `chmod 600` | Owner-only read/write — required by Ansible for cert files |
| Bridged adapter | VM gets a real network IP — reachable from WSL2 via the host |
| `.gitignore` | Prevents secrets from ever reaching GitHub |
| PAT | Personal Access Token — replaces GitHub passwords for Git operations |
| `--rebase` | Replays local commits on top of remote — keeps history linear |

---

*For detailed explanations of every component, see the full reference document.*