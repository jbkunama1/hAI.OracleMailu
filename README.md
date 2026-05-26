# 📬 Mailserver mit Mailu auf Oracle Cloud Free Tier

![Status](https://img.shields.io/badge/Status-Ready%20to%20Deploy-success?style=for-the-badge)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free-red?style=for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-Mailu%20Stack-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS%20%26%20Tunnel-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Debian](https://img.shields.io/badge/OS-Debian%20%2F%20Ubuntu-A81D33?style=for-the-badge&logo=debian&logoColor=white)

> Komplette Schritt-für-Schritt-Anleitung für einen eigenen Mailserver mit **Mailu**, **Oracle Cloud Always Free**, **Cloudflare DNS**, **Docker**, **Portainer** und **cloudflared**.

---

## 🗂️ Inhalt

1. [🎯 Zielbild & Architektur](#-zielbild--architektur)
2. [🧰 Voraussetzungen](#-voraussetzungen)
3. [🪪 Anmelden bei Oracle Cloud](#-anmelden-bei-oracle-cloud)
4. [🖥️ VM erstellen (Ampere A1 Flex)](#️-vm-erstellen-ampere-a1-flex)
5. [⚠️ Out of Capacity lösen](#️-out-of-capacity-lösen)
6. [🔐 Per SSH verbinden](#-per-ssh-verbinden)
7. [🌐 Oracle Security List konfigurieren](#-oracle-security-list-konfigurieren)
8. [⚙️ Linux Grundsystem vorbereiten](#️-linux-grundsystem-vorbereiten)
9. [🔥 iptables Firewall öffnen](#-iptables-firewall-öffnen)
10. [🐳 Docker & Portainer installieren](#-docker--portainer-installieren)
11. [☁️ cloudflared Tunnel einrichten](#️-cloudflared-tunnel-einrichten)
12. [📬 Mailu konfigurieren & starten](#-mailu-konfigurieren--starten)
13. [🌍 DNS in Cloudflare setzen](#-dns-in-cloudflare-setzen)
14. [📮 Port 25 bei Oracle freischalten](#-port-25-bei-oracle-freischalten)
15. [🧪 Testen & prüfen](#-testen--prüfen)
16. [🛠️ Troubleshooting](#️-troubleshooting)
17. [📊 RAM-Übersicht](#-ram-übersicht)

---

## 🎯 Zielbild & Architektur

```
Oracle Cloud Free Tier (ARM, Ampere A1)
└── Docker Engine
    ├── Portainer              → Admin UI (via cloudflared)
    ├── cloudflared            → Sicherer Tunnel für Admin-UIs
    └── Mailu Stack
        ├── Postfix            SMTP
        ├── Dovecot            IMAP
        ├── Rspamd             Spam-Filter
        ├── ClamAV             Virenscanner
        └── Roundcube          Webmailer (optional)
```

- 🔒 **Admin-UIs** sind nur via cloudflared Tunnel erreichbar (kein offener Port)
- 📨 **SMTP / IMAP** gehen direkt über die Oracle Public IP

---

## 🧰 Voraussetzungen

| Bereich | Empfehlung |
|---|---|
| Cloud | Oracle Cloud Always Free |
| Instanztyp | `VM.Standard.A1.Flex` (Ampere ARM) |
| CPU / RAM | 2 OCPU / 4 GB |
| OS | Debian 12 oder Ubuntu 22.04 LTS |
| Storage | 50 GB Boot Volume |
| Domain | z.B. `deinedomain.de` |
| DNS | Cloudflare |
| SSH-Key | Ed25519 empfohlen |

---

## 🪪 Anmelden bei Oracle Cloud

1. [https://cloud.oracle.com](https://cloud.oracle.com) öffnen und einloggen.
2. Im **OCI Dashboard** oben rechts die **Region** prüfen.
3. Frankfurt (`eu-frankfurt-1`) ist geografisch nah, aber oft ausgelastet.

> 💡 Noch kein Konto? [https://www.oracle.com/cloud/free/](https://www.oracle.com/cloud/free/) – Always Free benötigt eine Kreditkarte zur Verifizierung, es wird nichts berechnet.

---

## 🖥️ VM erstellen (Ampere A1 Flex)

### Schritt 1 – Compute öffnen

1. Links oben **☰ Menü → Compute → Instances**.
2. Auf **Create instance** klicken.

### Schritt 2 – Name & Compartment

- **Name:** `mail-server`
- **Compartment:** Standard oder eigenes

### Schritt 3 – Image & Shape

1. Bei **Image and shape** auf **Edit** klicken.
2. **Image:** Debian 12 oder Ubuntu 22.04.
3. **Shape → Ampere → VM.Standard.A1.Flex**:
   - OCPU: **2**
   - RAM: **4 GB**

### Schritt 4 – Netzwerk

- **VCN:** neu anlegen oder vorhandene nutzen.
- **Subnet:** Public Subnet.
- **Public IPv4:** ✅ aktivieren.

### Schritt 5 – SSH-Key

- Option **Paste public keys** wählen.
- Inhalt von `~/.ssh/id_ed25519.pub` einfügen.

### Schritt 6 – Erstellen

- **Create** klicken und warten bis Status **Running**.
- Öffentliche IP notieren.

---

## ⚠️ Out of Capacity lösen

| Lösung | Aufwand | Erfolgsrate |
|---|---|---|
| Später erneut versuchen | Niedrig | Mittel |
| Andere Availability Domain | Niedrig | Mittel |
| Auf **Pay As You Go** upgraden, nur Free-Ressourcen nutzen | Mittel | Hoch |
| [Retry-Script](https://github.com/hitrov/oci-arm-host-capacity) | Mittel | Sehr hoch |

> 💡 **Empfehlung:** Pay As You Go Upgrade + Budget-Alert auf 1 $ setzen. Oracle schaltet oft sofort Kapazität frei. Free-Ressourcen bleiben kostenlos.

---

## 🔐 Per SSH verbinden

```bash
chmod 600 ~/.ssh/id_ed25519
ssh -i ~/.ssh/id_ed25519 ubuntu@<PUBLIC_IP>
```

| Image | Benutzername |
|---|---|
| Ubuntu | `ubuntu` |
| Debian | `debian` |
| Oracle Linux | `opc` |

---

## 🌐 Oracle Security List konfigurieren

Im OCI-Webinterface:

1. Zur Instanz → **Subnet** klicken → **Security List** öffnen.
2. Unter **Ingress Rules** hinzufügen (Source: `0.0.0.0/0`, Protocol: TCP):

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

## ⚙️ Linux Grundsystem vorbereiten

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim git curl wget ca-certificates gnupg lsb-release iptables-persistent
sudo hostnamectl set-hostname mail.deinedomain.de
echo "127.0.0.1 mail.deinedomain.de mail localhost" | sudo tee -a /etc/hosts
```

---

## 🔥 iptables Firewall öffnen

```bash
sudo vim /etc/iptables/rules.v4
```

Inhalt:

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

## 🐳 Docker & Portainer installieren

### Docker

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
# Neu einloggen danach!
```

### Portainer

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

## ☁️ cloudflared Tunnel einrichten

```bash
cloudflared tunnel login
cloudflared tunnel create mail-admin
```

`/etc/cloudflared/config.yml`:

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
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

## 📬 Mailu konfigurieren & starten

### 1. Verzeichnis anlegen

```bash
sudo mkdir -p /mailu
sudo chown $USER:$USER /mailu
cd /mailu
```

### 2. Konfiguration erzeugen

[https://setup.mailu.io](https://setup.mailu.io) im Browser öffnen:

- Flavor: **Docker Compose**
- Domain: `deinedomain.de`
- Hostname: `mail.deinedomain.de`
- TLS: **Let's Encrypt**
- Komponenten: ✅ Postfix, Dovecot, Rspamd, ClamAV, optional Roundcube

`docker-compose.yml` und `mailu.env` nach `/mailu` kopieren.

### 3. Stack starten

```bash
cd /mailu
docker compose pull
docker compose up -d
docker ps
```

### 4. Admin-User anlegen

```bash
docker compose exec admin flask mailu admin \
  admin deinedomain.de 'SuperGeheimesPasswort'
```

---

## 🌍 DNS in Cloudflare setzen

> ⚠️ Der `mail`-Record muss **graue Wolke** (DNS only, kein Proxy) haben!

| Typ | Name | Wert | Hinweis |
|---|---|---|---|
| A | `mail` | `<Oracle IP>` | Proxy **aus** |
| MX | `@` | `mail.deinedomain.de` | Prio 10 |
| TXT | `@` | `v=spf1 mx ~all` | SPF |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; pct=25` | DMARC |
| TXT | `dkim._domainkey` | aus Mailu Admin | DKIM |

DKIM aus Mailu Admin holen: Domain öffnen → DKIM-Key anzeigen → in Cloudflare eintragen.

---

## 📮 Port 25 bei Oracle freischalten

1. Oracle Support öffnen → Ticket anlegen.
2. Betreff: **SMTP Port 25 Unblock Request**.
3. Begründung: privater Mailserver, SPF/DKIM/DMARC aktiv, kein Spam/Bulk-Mailing.

> ⏳ Ohne Freischaltung gehen ausgehende Mails zu externen Providern nicht an.

---

## 🧪 Testen & prüfen

### Mailclient einrichten

| Dienst | Server | Port | Sicherheit |
|---|---|---|---|
| IMAP | `mail.deinedomain.de` | 993 | SSL/TLS |
| SMTP | `mail.deinedomain.de` | 587 | STARTTLS |

### Zustellbarkeit prüfen

```bash
# Testmail an mail-tester.com schicken und Score checken
# Logs live beobachten:
cd /mailu
docker compose logs -f front postfix rspamd
```

---

## 🛠️ Troubleshooting

| Problem | Ursache | Lösung |
|---|---|---|
| Out of Capacity | Ampere-Kapazität ausgelastet | Pay As You Go Upgrade, Retry-Script |
| Let's Encrypt schlägt fehl | Port 80/443 zu | Security List + iptables prüfen |
| SMTP/IMAP geht nicht | Ports zu oder Proxy an | Security List, iptables, Cloudflare-Proxy aus |
| Mails im Spam | PTR/DKIM fehlt | Reverse DNS, DKIM, DMARC nachrüsten |
| Port 25 ausgehend geblockt | Oracle Default | Support-Ticket stellen |

---

## 📊 RAM-Übersicht

| Dienst | RAM |
|---|---:|
| Mailu gesamt | ~1.500 MB |
| Portainer | ~100 MB |
| cloudflared | ~50 MB |
| System + Docker | ~400 MB |
| **Gesamt** | **~2.050 MB** ✅ |

✅ Passt auf eine 4-GB-A1-Instanz.

---

## ✅ Checkliste

- [ ] Oracle-VM läuft, Public IP notiert
- [ ] SSH funktioniert
- [ ] Security List konfiguriert (7 Ports)
- [ ] iptables konfiguriert
- [ ] Docker läuft
- [ ] Portainer erreichbar (via cloudflared)
- [ ] cloudflared aktiv
- [ ] Mailu Stack läuft
- [ ] DNS gesetzt (A, MX, SPF, DMARC, DKIM)
- [ ] Port 25 freigegeben
- [ ] Testmail erfolgreich (mail-tester.com Score ≥ 8/10)

---

*Setup erstellt mit ❤️ und KI-Unterstützung*
