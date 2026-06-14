# Carbonia Email Server — Installer Guide

**Mailcow Dockerized** এর উপর ভিত্তি করে তৈরি, Ubuntu 22.04 এর জন্য। একটি script চালালেই সম্পূর্ণ email server সেটআপ হয়ে যাবে — DNS configuration থেকে শুরু করে SSL পর্যন্ত।

---

## Requirements

| Item | Minimum |
|---|---|
| OS | Ubuntu 22.04 LTS |
| RAM | 2 GB |
| Disk | 10 GB free |
| CPU | 2 cores |
| Network | Public IP, port 25 খোলা |

> Proxmox LXC/VM উভয়েই কাজ করে। Bare metal-এও চলবে।

---

## Pre-requisites

Script run করার আগে নিচের কাজগুলো নিশ্চিত করো:

**1. Port 25 unblock করো**
অনেক VPS provider (Hetzner, DigitalOcean, Vultr) by default port 25 block রাখে। Support ticket করে unblock করাতে হবে।

**2. Reverse DNS (PTR) সেট করো**
VPS control panel থেকে server IP-এর PTR record `mail.yourdomain.com` এ সেট করো।

**3. DNS provider API access প্রস্তুত করো (Cloudflare ব্যবহার করলে)**
- Cloudflare → My Profile → API Tokens → Create Token
- Zone ID: Cloudflare → তোমার Domain → Overview → right sidebar

---

## Installation

```bash
# Script download করো
wget https://your-server/carbonia-mailcow-installer.sh

# Run করো
sudo bash carbonia-mailcow-installer.sh
```

---

## Installation Steps

Script চালু হলে ধাপে ধাপে নিচের কাজগুলো করবে:

### Step 1 — System Audit
Server-এর hardware, OS, installed services, এবং required ports চেক করবে।

```
  ✔  RAM: 4096MB  ✓
  ✔  Disk: 50GB free  ✓
  ✔  CPU: 4 cores  ✓
  ✔  All required ports are available
```

Conflict পেলে warn করবে এবং continue করতে confirm নেবে।

---

### Step 2 — Configuration Input

Script নিজেই জিজ্ঞেস করবে:

```
  ▸ Domain: carbonia.email
  ▸ IP [203.0.113.10]:
  ▸ Timezone [Asia/Dhaka]:
  ▸ Email [postmaster@carbonia.email]:
```

তারপর DNS provider বেছে নাও:

```
  1)  Cloudflare           (Automatic DNS + SSL)
  2)  Microsoft Azure DNS  (Automatic via Azure CLI)
  3)  Google Cloud DNS     (Automatic via gcloud)
  4)  Manual               (Provide DNS records yourself)
```

Cloudflare বেছে নিলে API Token ও Zone ID চাইবে।

---

### Step 3 — System Preparation

- APT packages update ও upgrade
- NTP time sync enable
- Hostname `mail.yourdomain.com` সেট

---

### Step 4 — Docker Install

Docker Engine ও Docker Compose plugin automatically install হবে।
আগে থেকে install থাকলে skip করবে।

---

### Step 5 — DNS Records Configure

Cloudflare বেছে থাকলে automatically নিচের records তৈরি হবে:

| Type | Name | Value |
|---|---|---|
| A | mail.carbonia.email | `<server IP>` |
| MX | carbonia.email | mail.carbonia.email (priority 10) |
| TXT | carbonia.email | `v=spf1 mx a:mail.carbonia.email ~all` |
| TXT | _dmarc.carbonia.email | `v=DMARC1; p=quarantine; rua=mailto:...` |
| CNAME | autodiscover.carbonia.email | mail.carbonia.email |
| CNAME | autoconfig.carbonia.email | mail.carbonia.email |

Manual বেছে নিলে এই records নিজে DNS panel-এ add করতে হবে।

---

### Step 6 — Mailcow Install & Start

```
  ➜  Cloning Mailcow repository...
  ➜  Generating mailcow.conf...
  ➜  Pulling Docker images (this will take several minutes)...
  ➜  Starting all containers...
  ➜  Waiting for services to initialize (45 seconds)...
```

মোট ~20-30 মিনিট সময় লাগতে পারে image pull করতে।

---

### Step 7 — Firewall (UFW)

নিচের ports automatically allow হবে:

| Port | Protocol | Service |
|---|---|---|
| 25 | TCP | SMTP |
| 80 | TCP | HTTP / ACME (Let's Encrypt) |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 443 | TCP | HTTPS / Webmail |
| 465 | TCP | SMTPS |
| 587 | TCP | Submission |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 4190 | TCP | Sieve (mail filters) |

---

### Step 8 — DKIM Key

Rspamd container থেকে DKIM key extract করে DNS-এ publish করবে।

Manual DNS হলে Admin Panel থেকে নিতে হবে:
`Admin Panel → Configuration → ARC/DKIM Keys`

---

## Installation Complete

```
  ╔══════════════════════════════════════════════════════════════════════╗
  ║     ✅   CARBONIA EMAIL SERVER INSTALLATION COMPLETE!               ║
  ╚══════════════════════════════════════════════════════════════════════╝

  Admin Panel  : https://mail.carbonia.email/admin
  Webmail      : https://mail.carbonia.email
  Username     : admin
  Password     : moohoo  ← CHANGE IMMEDIATELY
```

---

## Post-Installation Checklist

- [ ] Admin password পরিবর্তন করো — default `moohoo` রাখা যাবে না
- [ ] Domain add করো: `Admin Panel → Mail Setup → Domains`
- [ ] প্রথম mailbox তৈরি করো
- [ ] DKIM verify করো: `Admin Panel → Configuration → ARC/DKIM Keys`
- [ ] MX record check করো: `nslookup -type=MX carbonia.email`
- [ ] Deliverability test করো: https://mail-tester.com

---

## Log & Record Files

| File | কাজ |
|---|---|
| `/var/log/carbonia-mailcow-installer.log` | Full installation log |
| `/root/mailcow-info.txt` | Server info summary |
| `/root/mailcow-dkim.txt` | DKIM key backup (auto DNS হলে) |

---

## Useful Commands

```bash
# Mailcow directory
cd /opt/mailcow-dockerized

# সব container-এর status দেখো
docker compose ps

# Container restart করো
docker compose restart

# Log দেখো
docker compose logs -f --tail=50

# Mailcow update করো
./update.sh
```

---

## Ports Checked During Audit

Script install-এর আগে এই ports blocked আছে কিনা check করে:
`25, 80, 110, 143, 443, 465, 587, 993, 995, 4190`

Conflict পেলে conflicting service (postfix, exim4, nginx ইত্যাদি) stop ও disable করে দেবে।

---

## Supported DNS Providers

| Provider | Method |
|---|---|
| Cloudflare | API Token (automatic) |
| Microsoft Azure DNS | Azure CLI (automatic) |
| Google Cloud DNS | gcloud CLI (automatic) |
| Manual | নিজে DNS panel-এ add করতে হবে |

---

## Troubleshooting

**Container উঠছে না?**
```bash
cd /opt/mailcow-dockerized
docker compose logs postfix-mailcow
docker compose logs rspamd-mailcow
```

**Port 25 blocked?**
VPS provider-এর support ticket করে port 25 unblock করাও।

**SSL certificate আসছে না?**
Port 80 এবং DNS A record সঠিক আছে কিনা check করো। Let's Encrypt এর জন্য port 80 খোলা থাকতে হবে।

**Mail spam হচ্ছে?**
- DKIM সঠিকভাবে configure হয়েছে কিনা দেখো
- PTR/rDNS সেট আছে কিনা দেখো
- https://mail-tester.com এ test করো
