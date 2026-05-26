# 📬 Mailu Mail Server on Oracle Cloud Free Tier

![Status](https://img.shields.io/badge/Status-Ready%20to%20Deploy-success?style=for-the-badge)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free-red?style=for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-Mailu%20Stack-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20%26%20Tunnel-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Debian](https://img.shields.io/badge/OS-Debian%20%2F%20Ubuntu-A81D33?style=for-the-badge&logo=debian&logoColor=white)

> A complete step-by-step guide to run your own mail server using **Mailu**, **Oracle Cloud Always Free**, **Cloudflare DNS**, **Docker**, **Portainer**, and **cloudflared**.

---

## 🗂️ Contents

1. [🎯 Target Architecture](#-target-architecture)
2. [🧰 Requirements](#-requirements)
3. [🪪 Sign in to Oracle Cloud](#-sign-in-to-oracle-cloud)
4. [🖥️ Create the VM (Ampere A1 Flex)](#️-create-the-vm-ampere-a1-flex)
5. [⚠️ Solve Out of Capacity](#️-solve-out-of-capacity)
6. [🔐 Connect via SSH](#-connect-via-ssh)
7. [🌐 Configure Oracle Security List](#-configure-oracle-security-list)
8. [⚙️ Prepare the Linux System](#️-prepare-the-linux-system)
9. [🔥 Open iptables Firewall](#-open-iptables-firewall)
10. [🐳 Install Docker & Portainer](#-install-docker--portainer)
11. [☁️ Set up cloudflared Tunnel](#️-set-up-cloudflared-tunnel)
12. [📬 Configure & Start Mailu](#-configure--start-mailu)
13. [🌍 Configure DNS in Cloudflare](#-configure-dns-in-cloudflare)
14. [📮 Request Port 25 Unblock](#-request-port-25-unblock)
15. [🧪 Test & Validate](#-test--validate)
16. [🛠️ Troubleshooting](#️-troubleshooting)
17. [📊 RAM Overview](#-ram-overview)

---

## 🎯 Target Architecture

```
Oracle Cloud Free Tier (ARM, Ampere A1)
└── Docker Engine
    ├── Portainer              → Admin UI (via cloudflared)
    ├── cloudflared            → Secure tunnel for admin UIs
    └── Mailu Stack
        ├── Postfix            SMTP
        ├── Dovecot            IMAP
        ├── Rspamd             Spam filter
        ├── ClamAV             Virus scanner
        └── Roundcube          Webmailer (optional)
```

- 🔒 **Admin UIs** only reachable via cloudflared tunnel
- 📨 **SMTP / IMAP** run directly over the Oracle public IP

---

## 🧰 Requirements

| Area | Recommendation |
|---|---|
| Cloud | Oracle Cloud Always Free |
| Instance type | `VM.Standard.A1.Flex` (Ampere ARM) |
| CPU / RAM | 2 OCPU / 4 GB |
| OS | Debian 12 or Ubuntu 22.04 LTS |
| Storage | 50 GB boot volume |
| Domain | e.g. `yourdomain.com` |
| DNS | Cloudflare |
| SSH Key | Ed25519 recommended |

---

## 🪪 Sign in to Oracle Cloud

1. Open [https://cloud.oracle.com](https://cloud.oracle.com) and log in.
2. Check the selected **region** in the top-right corner.
3. Frankfurt is close but often overloaded.

> 💡 No account? Sign up at [https://www.oracle.com/cloud/free/](https://www.oracle.com/cloud/free/)

---

## 🖥️ Create the VM (Ampere A1 Flex)

### Step 1 – Open Compute

1. **☰ Menu → Compute → Instances**.
2. Click **Create instance**.

### Step 2 – Image & Shape

1. Click **Edit** under **Image and shape**.
2. **Image:** Debian 12 or Ubuntu 22.04.
3. **Shape → Ampere → VM.Standard.A1.Flex**: 2 OCPU, 4 GB RAM.

### Step 3 – Networking

- Public Subnet, enable **Assign public IPv4 address**.

### Step 4 – SSH Key

- **Paste public keys** → paste `~/.ssh/id_ed25519.pub`.

### Step 5 – Create

- Click **Create**, wait for **Running** status, note the public IP.

---

## ⚠️ Solve Out of Capacity

| Solution | Effort | Success rate |
|---|---|---|
| Try again later | Low | Medium |
| Try another availability domain | Low | Medium |
| Upgrade to Pay As You Go, use only free resources | Medium | High |
| [Retry script](https://github.com/hitrov/oci-arm-host-capacity) | Medium | Very high |

---

## 🔐 Connect via SSH

```bash
chmod 600 ~/.ssh/id_ed25519
ssh -i ~/.ssh/id_ed25519 ubuntu@<PUBLIC_IP>
```

---

## 🌐 Configure Oracle Security List

Add Ingress Rules (TCP, Source `0.0.0.0/0`): ports 22, 25, 80, 443, 465, 587, 993.

---

## ⚙️ Prepare the Linux System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim git curl wget ca-certificates gnupg lsb-release iptables-persistent
sudo hostnamectl set-hostname mail.yourdomain.com
echo "127.0.0.1 mail.yourdomain.com mail localhost" | sudo tee -a /etc/hosts
```

---

## 🔥 Open iptables Firewall

```bash
sudo vim /etc/iptables/rules.v4
```

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p tcp --dport 22 -j ACCEPT
-A INPUT -p tcp --dport 25 -j ACCEPT
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
-A INPUT -p tcp --dport 465 -j ACCEPT
-A INPUT -p tcp --dport 587 -j ACCEPT
-A INPUT -p tcp --dport 993 -j ACCEPT
COMMIT
```

```bash
sudo iptables-restore < /etc/iptables/rules.v4
```

---

## 🐳 Install Docker & Portainer

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

```bash
docker run -d -p 127.0.0.1:9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/portainer:/data portainer/portainer-ce:latest
```

---

## ☁️ Set up cloudflared Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create mail-admin
```

`/etc/cloudflared/config.yml`:

```yaml
tunnel: mail-admin
credentials-file: /etc/cloudflared/mail-admin.json
ingress:
  - hostname: portainer.yourdomain.com
    service: https://localhost:9443
  - hostname: admin.yourdomain.com
    service: http://localhost:8080
  - hostname: webmail.yourdomain.com
    service: http://localhost:80
  - service: http_status:404
```

```bash
sudo systemctl enable cloudflared && sudo systemctl start cloudflared
```

---

## 📬 Configure & Start Mailu

```bash
sudo mkdir -p /mailu && sudo chown $USER:$USER /mailu && cd /mailu
```

Open [https://setup.mailu.io](https://setup.mailu.io), generate `docker-compose.yml` and `mailu.env`, copy to `/mailu`.

```bash
cd /mailu && docker compose pull && docker compose up -d
docker compose exec admin flask mailu admin admin yourdomain.com 'SuperSecretPassword'
```

---

## 🌍 Configure DNS in Cloudflare

> ⚠️ The `mail` record must use **gray cloud** (DNS only, no proxy)!

| Type | Name | Value | Note |
|---|---|---|---|
| A | `mail` | `<Oracle IP>` | Proxy **off** |
| MX | `@` | `mail.yourdomain.com` | Priority 10 |
| TXT | `@` | `v=spf1 mx ~all` | SPF |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; pct=25` | DMARC |
| TXT | `dkim._domainkey` | from Mailu Admin | DKIM |

---

## 📮 Request Port 25 Unblock

Open an Oracle Support ticket: **SMTP Port 25 Unblock Request**.
Explain: private server, SPF/DKIM/DMARC active, no spam/bulk mail.

---

## 🧪 Test & Validate

| Service | Server | Port | Security |
|---|---|---|---|
| IMAP | `mail.yourdomain.com` | 993 | SSL/TLS |
| SMTP | `mail.yourdomain.com` | 587 | STARTTLS |

```bash
cd /mailu && docker compose logs -f front postfix rspamd
```

Send a test mail to [https://mail-tester.com](https://mail-tester.com) and check your score.

---

## 🛠️ Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Out of Capacity | Ampere capacity full | Pay As You Go, retry script |
| Let's Encrypt fails | Port 80/443 blocked | Security List + iptables |
| SMTP/IMAP broken | Ports blocked / proxy on | Security List, iptables, disable CF proxy |
| Mail goes to spam | PTR/DKIM missing | Set reverse DNS, enable DKIM/DMARC |
| Port 25 blocked | Oracle default | Open support ticket |

---

## 📊 RAM Overview

| Service | RAM |
|---|---:|
| Mailu total | ~1,500 MB |
| Portainer | ~100 MB |
| cloudflared | ~50 MB |
| System + Docker | ~400 MB |
| **Total** | **~2,050 MB** ✅ |

---

## ✅ Checklist

- [ ] Oracle VM running, public IP noted
- [ ] SSH working
- [ ] Security List configured (7 ports)
- [ ] iptables configured
- [ ] Docker running
- [ ] Portainer reachable (via cloudflared)
- [ ] cloudflared active
- [ ] Mailu stack running
- [ ] DNS configured (A, MX, SPF, DMARC, DKIM)
- [ ] Port 25 unblocked
- [ ] Test mail successful (mail-tester.com score ≥ 8/10)

---

*Setup created with ❤️ and AI assistance*
