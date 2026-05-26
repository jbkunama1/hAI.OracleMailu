# 📬 Mail Server with Mailu on Oracle Cloud Free Tier

![Status](https://img.shields.io/badge/Status-Ready%20to%20Deploy-success?style=for-the-badge)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free-red?style=for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-Mailu%20Stack-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20%26%20Tunnel-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Debian](https://img.shields.io/badge/OS-Debian%20or%20Ubuntu-A81D33?style=for-the-badge&logo=debian&logoColor=white)

> A complete, structured guide to build your own mail server using **Mailu**, **Oracle Cloud Always Free**, **Cloudflare DNS**, **Docker**, **Portainer**, and **cloudflared**.

🇩🇪 [Deutsche Version → README.md](README.md)

---

## 🗂️ Contents

- [🎯 Target Architecture](#-target-architecture)
- [🧰 Requirements](#-requirements)
- [🪪 1. Sign in to Oracle Cloud](#-1-sign-in-to-oracle-cloud)
- [🖥️ 2. Create the VM in Oracle Cloud](#-2-create-the-vm-in-oracle-cloud)
- [🔐 3. Connect via SSH](#-3-connect-via-ssh)
- [🌐 4. Prepare Oracle Networking](#-4-prepare-oracle-networking)
- [⚙️ 5. Prepare the Linux System](#-5-prepare-the-linux-system)
- [🔥 6. Open the VM Firewall](#-6-open-the-vm-firewall)
- [🐳 7. Install Docker and Portainer](#-7-install-docker-and-portainer)
- [☁️ 8. Configure cloudflared Tunnel](#-8-configure-cloudflared-tunnel)
- [📬 9. Configure and Start Mailu](#-9-configure-and-start-mailu)
- [🌍 10. Configure DNS in Cloudflare](#-10-configure-dns-in-cloudflare)
- [📮 11. Request Port 25 Unblock](#-11-request-port-25-unblock)
- [🧪 12. Test and Validate](#-12-test-and-validate)
- [🛠️ 13. Troubleshooting](#-13-troubleshooting)
- [📊 Resource Overview](#-resource-overview)

---

## 🎯 Target Architecture

![Architecture](assets/architecture.png)

- ☁️ **Oracle Cloud Free Tier** with ARM VM `VM.Standard.A1.Flex`
- 🐳 **Docker** as runtime platform
- 📦 **Portainer** for stack management
- 📬 **Mailu** as the mail server stack
- 🔒 **cloudflared** to protect admin UIs
- 🌐 **Cloudflare DNS** for A, MX, SPF, DKIM, DMARC
- 📨 Direct mail transport via SMTP / IMAP

---

## 🧰 Requirements

| Area | Recommendation |
|---|---|
| Instance type | `VM.Standard.A1.Flex` |
| CPU / RAM | 2 OCPU / 4 GB RAM |
| OS | Debian 12 or Ubuntu LTS |
| Storage | 50 GB boot volume |
| Mail host | `mail.yourdomain.com` |
| DNS | Cloudflare, proxy **disabled** on mail record |

---

## 🪪 1. Sign in to Oracle Cloud

1. Open [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/).
2. Sign in with your Oracle account.
3. You will reach the **OCI Dashboard**.

> 💡 **Tip:** Frankfurt may be overloaded. Try another region for better A1 availability.

---

## 🖥️ 2. Create the VM in Oracle Cloud

### 2.1 Open Compute

1. Click **☰ menu** → **Compute → Instances**
2. Click **Create instance**

### 2.2 Image and Shape

1. Click **Edit** at **Image and shape**
2. Image: **Debian 12** or **Ubuntu LTS**
3. Shape: **Ampere** → **VM.Standard.A1.Flex**
4. Resources: **2 OCPU**, **4 GB RAM**

> ⚠️ **Out of Capacity?**
> - Try another availability domain
> - Upgrade to **Pay As You Go** (stays free for free-tier resources)
> - Use retry script: [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity)

### 2.3 Networking and SSH

1. Use a VCN with **public subnet**
2. Enable **Assign public IPv4 address**
3. Paste content of `~/.ssh/id_ed25519.pub`
4. Click **Create**, wait for **Running** status

---

## 🔐 3. Connect via SSH

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<PUBLIC_IP>
```

| Image | Default user |
|---|---|
| Ubuntu | `ubuntu` |
| Debian | `debian` |
| Oracle Linux | `opc` |

---

## 🌐 4. Prepare Oracle Networking

In OCI web console: Instance → Subnet → Security List → add **Ingress Rules**:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 25 | SMTP inbound |
| 80 | HTTP / Let's Encrypt |
| 443 | HTTPS |
| 465 | SMTPS |
| 587 | Submission |
| 993 | IMAPS |

---

## ⚙️ 5. Prepare the Linux System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim git curl wget ca-certificates gnupg lsb-release iptables-persistent
sudo hostnamectl set-hostname mail.yourdomain.com
echo "127.0.0.1 mail.yourdomain.com mail localhost" | sudo tee -a /etc/hosts
```

---

## 🔥 6. Open the VM Firewall

```bash
sudo vim /etc/iptables/rules.v4
```

```text
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

## 🐳 7. Install Docker and Portainer

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

> 🔁 Re-login once.

```bash
sudo mkdir -p /opt/portainer
docker run -d \
  -p 127.0.0.1:9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/portainer:/data \
  portainer/portainer-ce:latest
```

---

## ☁️ 8. Configure cloudflared Tunnel

File `/etc/cloudflared/config.yml`:

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

## 📬 9. Configure and Start Mailu

```bash
sudo mkdir -p /mailu && sudo chown $USER:$USER /mailu && cd /mailu
```

1. Open [https://setup.mailu.io](https://setup.mailu.io)
2. Flavor: **Docker Compose**
3. Main domain: `yourdomain.com` · Hostname: `mail.yourdomain.com`
4. TLS: **Let's Encrypt**
5. Services: ✅ Postfix ✅ Dovecot ✅ Rspamd ✅ ClamAV ➕ Roundcube optional
6. Copy `docker-compose.yml` and `mailu.env` to `/mailu`

```bash
cd /mailu
docker compose pull
docker compose up -d
docker ps
```

Create admin user:

```bash
docker compose exec admin flask mailu admin admin yourdomain.com 'SuperSecretPassword'
```

---

## 🌍 10. Configure DNS in Cloudflare

> ⚠️ The `mail` A-record must **NOT** be proxied → **gray cloud**!

| Type | Name | Value | Note |
|---|---|---|---|
| A | `mail` | `<Oracle Public IP>` | Proxy **off** |
| MX | `@` | `mail.yourdomain.com` | Priority 10 |
| TXT | `@` | `v=spf1 mx ~all` | SPF |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; pct=25` | DMARC |
| TXT | `dkim._domainkey` | from Mailu Admin | DKIM |

---

## 📮 11. Request Port 25 Unblock

Oracle Support → ticket → **SMTP Port 25 Unblock Request**  
Explain: private mail server, SPF/DKIM/DMARC active, no bulk mailing.

---

## 🧪 12. Test and Validate

| Service | Server | Port | Security |
|---|---|---|---|
| IMAP | `mail.yourdomain.com` | 993 | SSL/TLS |
| SMTP | `mail.yourdomain.com` | 587 | STARTTLS |

```bash
cd /mailu && docker compose logs -f front postfix rspamd
```

Deliverability test: [https://mail-tester.com](https://mail-tester.com)

---

## 🛠️ 13. Troubleshooting

| Problem | Fix |
|---|---|
| Out of Capacity | Switch AD, PAYG upgrade, retry script |
| Let's Encrypt fails | Port 80/443 open? DNS correct? Proxy off? |
| SMTP/IMAP broken | Security List, iptables, containers running? |
| Mail goes to spam | Set PTR, check SPF/DKIM/DMARC |

---

## 📊 Resource Overview

| Service | RAM |
|---|---:|
| Mailu total | ~1.5 GB |
| Portainer | ~100 MB |
| cloudflared | ~50 MB |
| System + Docker | ~400 MB |
| **Total** | **~2.0 GB** ✅ |

---

## ✅ Checklist

- [ ] Oracle VM running
- [ ] SSH works
- [ ] Security List configured
- [ ] iptables configured
- [ ] Docker running
- [ ] Portainer reachable
- [ ] cloudflared active
- [ ] Mailu running
- [ ] DNS configured
- [ ] Port 25 unblocked
- [ ] DKIM added
- [ ] Test mail successful
