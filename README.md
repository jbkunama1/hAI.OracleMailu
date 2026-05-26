# 📬 Mailserver mit Mailu auf Oracle Cloud Free Tier

![Status](https://img.shields.io/badge/Status-Ready%20to%20Deploy-success?style=for-the-badge)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free-red?style=for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-Mailu%20Stack-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20%26%20Tunnel-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Debian](https://img.shields.io/badge/OS-Debian%20or%20Ubuntu-A81D33?style=for-the-badge&logo=debian&logoColor=white)

> Eine komplette, strukturierte Anleitung für einen eigenen Mailserver mit **Mailu**, **Oracle Cloud Always Free**, **Cloudflare DNS**, **Docker**, **Portainer** und **cloudflared**.

🇬🇧 [English version → README_EN.md](README_EN.md)

---

## 🗂️ Inhalt

- [🎯 Zielbild](#-zielbild)
- [🧰 Voraussetzungen](#-voraussetzungen)
- [🪪 1. Anmelden bei Oracle Cloud](#-1-anmelden-bei-oracle-cloud)
- [🖥️ 2. VM in Oracle Cloud erstellen](#-2-vm-in-oracle-cloud-erstellen)
- [🔐 3. Per SSH verbinden](#-3-per-ssh-verbinden)
- [🌐 4. Oracle Netzwerk vorbereiten](#-4-oracle-netzwerk-vorbereiten)
- [⚙️ 5. Linux-Grundsystem vorbereiten](#-5-linux-grundsystem-vorbereiten)
- [🔥 6. Firewall auf der VM öffnen](#-6-firewall-auf-der-vm-öffnen)
- [🐳 7. Docker und Portainer installieren](#-7-docker-und-portainer-installieren)
- [☁️ 8. cloudflared Tunnel einrichten](#-8-cloudflared-tunnel-einrichten)
- [📬 9. Mailu konfigurieren und starten](#-9-mailu-konfigurieren-und-starten)
- [🌍 10. DNS in Cloudflare setzen](#-10-dns-in-cloudflare-setzen)
- [📮 11. Port 25 freischalten lassen](#-11-port-25-freischalten-lassen)
- [🧪 12. Testen und prüfen](#-12-testen-und-prüfen)
- [🛠️ 13. Troubleshooting](#-13-troubleshooting)
- [📊 Ressourcenübersicht](#-ressourcenübersicht)

---

## 🎯 Zielbild

![Architektur](assets/architecture.png)

Am Ende läuft dein Setup so:

- ☁️ **Oracle Cloud Free Tier** mit ARM-VM `VM.Standard.A1.Flex`
- 🐳 **Docker** als Laufzeitplattform
- 📦 **Portainer** für Stack-Verwaltung
- 📬 **Mailu** als Mailserver-Stack
- 🔒 **cloudflared** für Admin-UIs ohne offene Webports
- 🌐 **Cloudflare DNS** für A, MX, SPF, DKIM, DMARC
- 📨 Direkter Mailverkehr via SMTP / IMAP über die Oracle-IP

---

## 🧰 Voraussetzungen

| Bereich | Empfehlung |
|---|---|
| Instanztyp | `VM.Standard.A1.Flex` |
| CPU / RAM | 2 OCPU / 4 GB RAM |
| OS | Debian 12 oder Ubuntu LTS |
| Storage | 50 GB Boot Volume |
| Maildomain | `mail.deinedomain.de` |
| DNS | Cloudflare, Proxy beim Mail-Record **aus** |

---

## 🪪 1. Anmelden bei Oracle Cloud

1. Öffne [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/).
2. Melde dich mit deinem Oracle-Konto an.
3. Nach dem Login landest du im **OCI Dashboard**.

> 💡 **Tipp:** Frankfurt ist naheliegend, aber oft ausgelastet.

---

## 🖥️ 2. VM in Oracle Cloud erstellen

### 2.1 Compute öffnen

1. Auf **☰ Menü** klicken → **Compute → Instances**
2. Auf **Create instance** klicken

### 2.2 Image und Shape

1. Bei **Image and shape** auf **Edit** klicken
2. Image: **Debian 12** oder **Ubuntu LTS**
3. Shape: Kategorie **Ampere** → **VM.Standard.A1.Flex**
4. Ressourcen: **2 OCPU**, **4 GB RAM**

> ⚠️ **Out of Capacity?**
> - Availability Domain wechseln
> - Auf **Pay As You Go** upgraden (bleibt kostenlos bei Free-Ressourcen)
> - Retry-Script verwenden: [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity)

### 2.3 Netzwerk & SSH

1. VCN mit **Public Subnet** verwenden
2. **Assign public IPv4 address** aktivieren
3. SSH-Key einfügen: Inhalt von `~/.ssh/id_ed25519.pub`
4. Auf **Create** klicken, auf Status **Running** warten

---

## 🔐 3. Per SSH verbinden

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<PUBLIC_IP>
```

| Image | Benutzer |
|---|---|
| Ubuntu | `ubuntu` |
| Debian | `debian` |
| Oracle Linux | `opc` |

---

## 🌐 4. Oracle Netzwerk vorbereiten

Im OCI-Webinterface: Instanz → Subnet → Security List → **Ingress Rules** hinzufügen:

| Port | Zweck |
|---|---|
| 22 | SSH |
| 25 | SMTP Empfang |
| 80 | HTTP / Let's Encrypt |
| 443 | HTTPS |
| 465 | SMTPS |
| 587 | Submission |
| 993 | IMAPS |

---

## ⚙️ 5. Linux-Grundsystem vorbereiten

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim git curl wget ca-certificates gnupg lsb-release iptables-persistent
sudo hostnamectl set-hostname mail.deinedomain.de
echo "127.0.0.1 mail.deinedomain.de mail localhost" | sudo tee -a /etc/hosts
```

---

## 🔥 6. Firewall auf der VM öffnen

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

## 🐳 7. Docker und Portainer installieren

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

> 🔁 Danach neu einloggen.

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

## ☁️ 8. cloudflared Tunnel einrichten

Datei `/etc/cloudflared/config.yml`:

```yaml
tunnel: mail-admin
credentials-file: /etc/cloudflared/mail-admin.json

ingress:
  - hostname: portainer.deinedomain.de
    service: https://localhost:9443
  - hostname: admin.deinedomain.de
    service: http://localhost:8080
  - hostname: webmail.deinedomain.de
    service: http://localhost:80
  - service: http_status:404
```

```bash
sudo systemctl enable cloudflared && sudo systemctl start cloudflared
```

---

## 📬 9. Mailu konfigurieren und starten

```bash
sudo mkdir -p /mailu && sudo chown $USER:$USER /mailu && cd /mailu
```

1. Öffne [https://setup.mailu.io](https://setup.mailu.io)
2. Flavor: **Docker Compose**
3. Main domain: `deinedomain.de` · Hostname: `mail.deinedomain.de`
4. TLS: **Let's Encrypt** aktivieren
5. Dienste: ✅ Postfix ✅ Dovecot ✅ Rspamd ✅ ClamAV ➕ Roundcube optional
6. `docker-compose.yml` und `mailu.env` nach `/mailu` kopieren

```bash
cd /mailu
docker compose pull
docker compose up -d
docker ps
```

Admin-User anlegen:

```bash
docker compose exec admin flask mailu admin admin deinedomain.de 'SuperGeheimesPasswort'
```

---

## 🌍 10. DNS in Cloudflare setzen

> ⚠️ Der `mail`-A-Record darf **nicht** über Cloudflare proxied sein → **graue Wolke**!

| Typ | Name | Wert | Hinweis |
|---|---|---|---|
| A | `mail` | `<Oracle Public IP>` | Proxy **aus** |
| MX | `@` | `mail.deinedomain.de` | Prio 10 |
| TXT | `@` | `v=spf1 mx ~all` | SPF |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; pct=25` | DMARC |
| TXT | `dkim._domainkey` | aus Mailu Admin | DKIM |

---

## 📮 11. Port 25 freischalten lassen

Oracle Support → Ticket → **SMTP Port 25 Unblock Request**  
Kurz begründen: privater Mailserver, SPF/DKIM/DMARC aktiv, kein Bulk Mailing.

---

## 🧪 12. Testen und prüfen

| Dienst | Server | Port | Sicherheit |
|---|---|---|---|
| IMAP | `mail.deinedomain.de` | 993 | SSL/TLS |
| SMTP | `mail.deinedomain.de` | 587 | STARTTLS |

```bash
cd /mailu && docker compose logs -f front postfix rspamd
```

Zustellbarkeit testen: [https://mail-tester.com](https://mail-tester.com)

---

## 🛠️ 13. Troubleshooting

| Problem | Lösung |
|---|---|
| Out of Capacity | Andere AD, PAYG-Upgrade, Retry-Script |
| Let's Encrypt fehlt | Port 80/443 offen? DNS korrekt? Proxy aus? |
| SMTP/IMAP geht nicht | Security List, iptables, Container laufen? |
| Mails im Spam | PTR setzen, SPF/DKIM/DMARC prüfen |

---

## 📊 Ressourcenübersicht

| Dienst | RAM |
|---|---:|
| Mailu gesamt | ~1.5 GB |
| Portainer | ~100 MB |
| cloudflared | ~50 MB |
| System + Docker | ~400 MB |
| **Gesamt** | **~2.0 GB** ✅ |

---

## ✅ Checkliste

- [ ] Oracle-VM läuft
- [ ] SSH funktioniert
- [ ] Security List gesetzt
- [ ] iptables gesetzt
- [ ] Docker läuft
- [ ] Portainer erreichbar
- [ ] cloudflared aktiv
- [ ] Mailu läuft
- [ ] DNS gesetzt
- [ ] Port 25 freigegeben
- [ ] DKIM eingetragen
- [ ] Testmail erfolgreich
