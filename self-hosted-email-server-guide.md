# Self-Hosted Email Server — Complete Project Documentation

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Platform](https://img.shields.io/badge/platform-Linux%20%2F%20Ubuntu%2022.04-orange.svg)
![Status](https://img.shields.io/badge/status-production%20%2F%20active-brightgreen.svg)
![Components](https://img.shields.io/badge/components-Postfix%20%2B%20Dovecot%20%2B%20Roundcube%20%2B%20Brevo%20%2B%20Cloudflare%20%2B%20Webmin-blueviolet.svg)
![Last Updated](https://img.shields.io/badge/last%20updated-2026--09--30-success.svg)
![Lines](https://img.shields.io/badge/lines-~1065-lightgrey.svg)
![Self-Hosted](https://img.shields.io/badge/self--hosted-yes-success.svg)
![Email](https://img.shields.io/badge/email-SMTP%20%2F%20IMAP%20%2F%20Webmail-informational.svg)
![Security](https://img.shields.io/badge/security-TLS%20%2F%20SPF%20%2F%20DKIM%20%2F%20DMARC-critical.svg)
![DNS](https://img.shields.io/badge/DNS-Cloudflare-blue.svg)
![Relay](https://img.shields.io/badge/outbound%20relay-Brevo-ff69b4.svg)
![Inbound](https://img.shields.io/badge/inbound-Cloudflare%20Worker-ff9900.svg)
![Webmail](https://img.shields.io/badge/webmail-Roundcube-9cf.svg)
![Management](https://img.shields.io/badge/management-Webmin-blue.svg)
![Auth](https://img.shields.io/badge/auth-SASL%20%2F%20SHA512--CRYPT-yellowgreen.svg)
![Storage](https://img.shields.io/badge/storage-Maildir-green.svg)
![Firewall](https://img.shields.io/badge/firewall-UFW-red.svg)
![Monitoring](https://img.shields.io/badge/monitoring-systemd%20%2F%20cron-yellow.svg)
![Backups](https://img.shields.io/badge/backups-tar%20%2F%20rsync%20%2F%20rclone-orange.svg)
![Testing](https://img.shields.io/badge/testing-mail--tester%20%2F%20swaks-blueviolet.svg)
![Uptime](https://img.shields.io/badge/uptime-99%25%2B-success.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)
![Contributors](https://img.shields.io/badge/contributors-1-blue.svg)
![Stars](https://img.shields.io/badge/stars-%E2%98%85%200-yellow.svg)
![Forks](https://img.shields.io/badge/forks-%E2%9C%85%200-lightgrey.svg)

A full-stack, self-hosted email solution on a Linux VPS: **Postfix** (SMTP), **Dovecot** (IMAP), **Roundcube** (webmail), **Brevo** (outbound relay), **Cloudflare Email Routing + Worker** (inbound catch-all), and **Webmin** (management).

> **Privacy note:** This document has been fully redacted for public sharing. All real domains, email addresses, IP addresses, hostnames, and personal names have been replaced with example placeholders (e.g. `example.com`, `john@example.com`, `192.168.1.100`, `John Doe`). Replace every placeholder with your own values before use.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Prerequisites](#3-prerequisites)
4. [Base Server Setup](#4-base-server-setup)
5. [DNS Configuration (Cloudflare)](#5-dns-configuration-cloudflare)
6. [Postfix — Inbound SMTP Server](#6-postfix--inbound-smtp-server)
7. [Dovecot — IMAP Server](#7-dovecot--imap-server)
8. [Roundcube — Webmail](#8-roundcube--webmail)
9. [The `addmail` Script](#9-the-addmail-script)
10. [Cloudflare Email Routing & Worker](#10-cloudflare-email-routing--worker)
11. [Brevo — Outbound Relay](#11-brevo--outbound-relay)
12. [Webmin — Server Management](#12-webmin--server-management)
13. [Managing Mail Users](#13-managing-mail-users)
14. [Aliases — Extra Addresses into an Existing Mailbox](#14-aliases--extra-addresses-into-an-existing-mailbox)
15. [Sending as a Different Address](#15-sending-as-a-different-address)
16. [Fixing the Display Name (From Header)](#16-fixing-the-display-name-from-header)
17. [Email Deliverability — SPF, DKIM, DMARC](#17-email-deliverability--spf-dkim-dmarc)
18. [Reputation & Content Best Practices](#18-reputation--content-best-practices)
19. [Troubleshooting](#19-troubleshooting)
20. [Maintenance & Operations](#20-maintenance--operations)
21. [Security Notes](#21-security-notes)
22. [Appendix — Full Configuration Files](#22-appendix--full-configuration-files)

---

## 1. Project Overview

This project turns a small Linux VPS into a complete email server for a personal domain. Instead of paying for Google Workspace or Microsoft 365, you run the mail stack yourself:

- **Inbound mail** arrives via Cloudflare Email Routing, which forwards every address at your domain to a Cloudflare Worker. The Worker POSTs the raw message to your server's inbound endpoint.
- **Postfix** receives the message, checks whether the recipient mailbox exists, and either delivers it to the virtual mailbox store or rejects it.
- **Dovecot** serves the stored mail over IMAP so any mail client (and Roundcube) can read it.
- **Roundcube** provides a browser-based webmail interface.
- **Outbound mail** is relayed through **Brevo** (Sendinblue), so you don't have to manage your own sending IP reputation or deal with port 25 blocking by residential ISPs.
- **Webmin** gives you a web-based GUI for server administration.

### Why this architecture?

| Problem | Solution |
|---|---|
| Port 25 often blocked on home connections | Relay outbound through Brevo's SMTP |
| Dynamic home IP | Cloudflare Worker accepts mail and forwards it over HTTPS |
| No static DNS for catch-all | Cloudflare Email Routing catch-all → Worker |
| Managing mail users by hand is error-prone | `addmail` script automates everything |
| Webmail needed | Roundcube |

---

## 2. Architecture Diagram

```
                         ┌─────────────────────────────┐
                         │        INTERNET              │
                         └──────────────┬──────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              │                         │                         │
              ▼                         ▼                         ▼
   ┌──────────────────┐    ┌───────────────────────┐    ┌────────────────┐
   │  Sending MTA     │    │  Cloudflare DNS       │    │  Recipient     │
   │  (Gmail, etc.)   │    │  + Email Routing      │    │  Mail Server   │
   └────────┬─────────┘    └───────────┬───────────┘    │  (your VPS)    │
            │                          │                │                │
            │  1. MX lookup            │  2. Catch-all  │                │
            │ ──────────────────────►  │  rule matches  │                │
            │                          │  every address │                │
            │                          │ ─────────────► │                │
            │                          │  3. Worker     │                │
            │                          │     POSTs raw  │                │
            │                          │     message    │                │
            │                          │ ─────────────► │  4. Postfix    │
            │                          │                │     receives    │
            │                          │                │  5. Dovecot     │
            │                          │                │     stores in   │
            │                          │                │     vmail       │
            │                          │                │  6. Roundcube / │
            │                          │                │     IMAP client │
            │                          │                │     reads it    │
            │                          │                └────────────────┘
            │                          │
            │  7. Outbound: Roundcube  │
            │     → Postfix → Brevo    │
            │     SMTP relay ──────────┼──────► Brevo ──► recipient
            └──────────────────────────┘
```

### Inbound flow (someone emails you)

1. Sender's mail server looks up the MX records for `example.com`.
2. Cloudflare's Email Routing catch-all rule matches **any** address at the domain.
3. Cloudflare forwards the message to a **Worker** (a small serverless function).
4. Worker POSTs the raw RFC 5322 message to `https://mail.example.com/inbound` (your server).
5. Postfix receives it, checks `/etc/postfix/vmailbox` for the recipient.
6. If the mailbox exists → deliver to `/var/mail/vhosts/example.com/<user>/`. If not → reject with "User unknown".
7. Dovecot makes the mail available over IMAP; Roundcube or any client reads it.

### Outbound flow (you email someone)

1. You compose in Roundcube or any mail client.
2. Postfix accepts the message on the submission port (587).
3. Postfix relays it to Brevo's SMTP server (`smtp-relay.brevo.com:587`) using your Brevo SMTP credentials.
4. Brevo signs the message with DKIM (using your domain) and delivers it to the recipient.

---

## 3. Prerequisites

### Server

- A Linux VPS (Ubuntu 22.04 LTS recommended) with a **public static IP**.
- At least 1 GB RAM, 10 GB disk.
- Root or sudo access.
- Ports **25** (SMTP), **587** (submission), **993** (IMAPS), **443** (HTTPS) open in the firewall and not blocked by the provider.

### Domain

- A domain registered with any registrar, using **Cloudflare** as the DNS provider (free plan is fine).
- The domain must be added to Cloudflare and the nameservers switched.

### Accounts

- A **Brevo** account (free tier: 300 emails/day) for outbound relay.
- A **Cloudflare** account with the domain added.

### Placeholders used throughout this document

| Placeholder | Meaning | Replace with |
|---|---|---|
| `example.com` | Your domain | Your actual domain |
| `mail.example.com` | Mail server hostname (A record) | Your server's public IP |
| `webmail.example.com` | Roundcube hostname | Your server's public IP |
| `john@example.com` | Primary mailbox | Your real address |
| `jane@example.com` | Secondary mailbox | Real address |
| `info@example.com` | Alias target | Real address |
| `postmaster@example.com` | DMARC reports address | Real address |
| `192.168.1.100` | Server's private IP (example) | Your server's actual IP |
| `John Doe` | Display name | Your real name |
| `your-brevo-smtp-login` | Brevo SMTP username | From Brevo dashboard |
| `your-brevo-smtp-key` | Brevo SMTP password | From Brevo dashboard |

---

## 4. Base Server Setup

### 4.1 Set the hostname

```bash
sudo hostnamectl set-hostname mail.example.com
```

Edit `/etc/hosts` so it looks like this (use your server's **public** IP, not the private one):

```text
127.0.0.1       localhost
203.0.113.10    mail.example.com mail
192.168.1.100   mail.example.com mail
```

> **Important:** The `Received:` header in every outgoing message is built from the hostname. A stale or placeholder hostname (e.g. `mail.example.co.za` from an earlier draft) leaks into headers and looks unprofessional. Always verify:
>
> ```bash
> hostname
> getent hosts $(hostname)
> ```

### 4.2 Update the system

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git ufw
```

### 4.3 Configure the firewall

```bash
sudo ufw allow 25/tcp      # SMTP (inbound)
sudo ufw allow 587/tcp     # Submission (outbound relay)
sudo ufw allow 993/tcp     # IMAPS
sudo ufw allow 443/tcp     # HTTPS (Roundcube, Worker endpoint)
sudo ufw allow 80/tcp      # HTTP (Let's Encrypt challenge)
sudo ufw enable
```

### 4.4 Set the mail name

```bash
echo "example.com" | sudo tee /etc/mailname
```

---

## 5. DNS Configuration (Cloudflare)

In the Cloudflare dashboard, go to **DNS → Records** and create the following:

### 5.1 A records

| Type | Name | Content | Proxy status |
|---|---|---|---|
| A | `mail` | `203.0.113.10` | DNS only (grey cloud) |
| A | `webmail` | `203.0.113.10` | DNS only (grey cloud) |

### 5.2 MX record (for Email Routing)

| Type | Name | Content | Priority |
|---|---|---|---|
| MX | `@` | `route1.mx.cloudflare.net` | 10 |
| MX | `@` | `route2.mx.cloudflare.net` | 10 |
| MX | `@` | `route3.mx.cloudflare.net` | 10 |

> These MX records point to Cloudflare's Email Routing servers, **not** your own server. Cloudflare receives the mail first, then forwards it via the Worker.

### 5.3 SPF record

Exactly **one** TXT record starting with `v=spf1`:

```text
v=spf1 include:_spf.mx.cloudflare.net include:spf.brevo.com ~all
```

- `include:_spf.mx.cloudflare.net` — allows Cloudflare Email Routing to send on your behalf.
- `include:spf.brevo.com` — allows Brevo to send on your behalf.
- `~all` — soft-fail everything else (use `-all` once you're confident).

> **Never create two SPF records.** Multiple `v=spf1` records cause a permanent authentication error. If you need to add another sender, merge it into the single record.

### 5.4 DKIM record

DKIM is provided by Brevo. In the Brevo dashboard:

1. Go to **Senders, Domains & Dedicated IPs → Domains**.
2. Select `example.com`.
3. Click **Authenticate**.
4. Brevo shows a DKIM record (usually a TXT record named `brevo2._domainkey` or similar).

Add it in Cloudflare exactly as shown:

| Type | Name | Content | Proxy status |
|---|---|---|---|
| TXT | `brevo2._domainkey` | `v=DKIM1; k=rsa; p=...` | DNS only |

> If Brevo gives you a CNAME for DKIM instead of a TXT, set the proxy status to **DNS only** (grey cloud) — proxied CNAMEs break DKIM lookups.

### 5.5 DMARC record

| Type | Name | Content |
|---|---|---|
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:postmaster@example.com` |

- `p=none` — monitor only, no enforcement. Move to `p=quarantine` after a few weeks of clean passes, then `p=reject` eventually.
- `rua=mailto:postmaster@example.com` — aggregate reports are sent to this address. **You must create the `postmaster@example.com` alias** (see [Section 14](#14-aliases--extra-addresses-into-an-existing-mailbox)) or the reports bounce.

### 5.6 Verify in Cloudflare

Cloudflare's Email Routing page will show the domain as **Active** once the MX records propagate. This can take a few minutes to a few hours.

---

## 6. Postfix — Inbound SMTP Server

### 6.1 Install

```bash
sudo apt install -y postfix
```

During installation, choose **Internet Site** and set the System mail name to `example.com`.

### 6.2 Main configuration

Edit `/etc/postfix/main.cf`:

```ini
# ==========================================================================
# BASIC SETTINGS
# ==========================================================================

smtpd_banner = $myhostname ESMTP
compatibility_level = 2

# ==========================================================================
# HOSTNAME / DOMAIN
# ==========================================================================

myhostname = mail.example.com
mydomain = example.com
myorigin = $mydomain
mydestination = localhost.$mydomain, localhost
mynetworks = 127.0.0.0/8

# ==========================================================================
# VIRTUAL MAILBOX SETTINGS
# ==========================================================================

virtual_mailbox_domains = example.com
virtual_mailbox_base = /var/mail/vhosts
virtual_mailbox_maps = hash:/etc/postfix/vmailbox
virtual_minimum_uid = 1000
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

# ==========================================================================
# ALIASES
# ==========================================================================

virtual_alias_maps = hash:/etc/postfix/virtual_aliases

# ==========================================================================
# TLS
# ==========================================================================

smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_loglevel = 1

smtp_tls_security_level = may
smtp_tls_loglevel = 1

# ==========================================================================
# SASL AUTHENTICATION (for submission)
# ==========================================================================

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname

# ==========================================================================
# RESTRICTIONS
# ==========================================================================

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_invalid_hostname,
    reject_non_fqdn_hostname,
    reject_non_fqdn_sender,
    reject_non_fqdn_recipient,
    reject_unknown_sender_domain,
    reject_unknown_recipient_domain,
    reject_rbl_client zen.spamhaus.org,
    permit

smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    defer_unauth_destination

# ==========================================================================
# RELAY THROUGH BREVO (outbound)
# ==========================================================================

relayhost = [smtp-relay.brevo.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_tls_security_level = encrypt

# ==========================================================================
# MESSAGE SIZE & TIMEOUTS
# ==========================================================================

message_size_limit = 26214400
mailbox_size_limit = 0
```

### 6.3 Master configuration

Edit `/etc/postfix/master.cf` to enable the submission port:

```ini
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

### 6.4 Create the virtual mailbox map

```bash
sudo mkdir -p /var/mail/vhosts/example.com
sudo groupadd -g 5000 vmail 2>/dev/null || true
sudo useradd -g vmail -u 5000 -d /var/mail -s /usr/sbin/nologin vmail 2>/dev/null || true
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

Create `/etc/postfix/vmailbox`:

```text
john@example.com    example.com/john/
jane@example.com    example.com/jane/
```

Build the hash database:

```bash
sudo postmap /etc/postfix/vmailbox
```

### 6.5 Create the Brevo SASL password file

Create `/etc/postfix/sasl_passwd`:

```text
[smtp-relay.brevo.com]:587    your-brevo-smtp-login:your-brevo-smtp-key
```

```bash
sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

### 6.6 Create the virtual aliases file

```bash
sudo touch /etc/postfix/virtual_aliases
sudo postmap /etc/postfix/virtual_aliases
```

### 6.7 Start Postfix

```bash
sudo systemctl enable postfix
sudo systemctl restart postfix
sudo systemctl status postfix
```

---

## 7. Dovecot — IMAP Server

### 7.1 Install

```bash
sudo apt install -y dovecot-imapd dovecot-lmtpd
```

### 7.2 Main configuration

Edit `/etc/dovecot/dovecot.conf`:

```ini
protocols = imap lmtp
listen = *, ::
```

### 7.3 Mail location

Edit `/etc/dovecot/conf.d/10-mail.conf`:

```ini
mail_location = maildir:/var/mail/vhosts/%d/%n
mail_uid = 5000
mail_gid = 5000
mail_privileged_group = vmail
```

### 7.4 Authentication

Edit `/etc/dovecot/conf.d/10-auth.conf`:

```ini
disable_plaintext_auth = yes
auth_mechanisms = login plain
!include auth-passwdfile.conf.ext
```

Create `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`:

```ini
passdb {
  driver = passwd-file
  args = scheme=SHA512-CRYPT /etc/dovecot/users
}

userdb {
  driver = static
  args = uid=5000 gid=5000 home=/var/mail/vhosts/%d/%n
}
```

### 7.5 Create the users file

Create `/etc/dovecot/users` (generate the hash with `doveadm pw -s SHA512-CRYPT`):

```text
john@example.com:{SHA512-CRYPT}$6$rounds=50000$REPLACE_WITH_REAL_SALT$REPLACE_WITH_REAL_HASH
jane@example.com:{SHA512-CRYPT}$6$rounds=50000$REPLACE_WITH_REAL_SALT$REPLACE_WITH_REAL_HASH
```

Set correct permissions:

```bash
sudo chown root:dovecot /etc/dovecot/users
sudo chmod 640 /etc/dovecot/users
```

### 7.6 Enable the LMTP socket for Postfix delivery

Edit `/etc/dovecot/conf.d/10-master.conf` and make sure the `unix_listener /var/spool/postfix/private/dovecot-lmtp` block exists inside the `service lmtp` section:

```ini
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

Also enable the auth socket for Postfix SASL:

```ini
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

### 7.7 Start Dovecot

```bash
sudo systemctl enable dovecot
sudo systemctl restart dovecot
sudo systemctl status dovecot
```

---

## 8. Roundcube — Webmail

### 8.1 Install

```bash
sudo apt install -y roundcube roundcube-mysql roundcube-plugins
```

During installation, choose to configure the database with `dbconfig-common`. Set a strong database password.

### 8.2 Configure Apache/Nginx

For Apache:

```bash
sudo ln -s /etc/roundcube/apache.conf /etc/apache2/conf-enabled/roundcube.conf
sudo systemctl restart apache2
```

For Nginx, create a server block pointing to `/usr/share/roundcube` and enable PHP-FPM.

### 8.3 Enable HTTPS

```bash
sudo apt install -y certbot python3-certbot-apache   # or python3-certbot-nginx
sudo certbot --apache -d webmail.example.com         # or --nginx
```

### 8.4 Configure Roundcube

Edit `/etc/roundcube/config.inc.php`:

```php
$config['default_host'] = 'ssl://localhost';
$config['default_port'] = 993;
$config['smtp_server'] = 'tls://localhost';
$config['smtp_port'] = 587;
$config['smtp_user'] = '%u';
$config['smtp_pass'] = '%p';
$config['product_name'] = 'Webmail';
$config['des_key'] = 'REPLACE_WITH_24_CHAR_RANDOM_STRING';
$config['plugins'] = ['archive', 'zipdownload', 'managesieve'];
$config['language'] = 'en_US';
$config['skin'] = 'elastic';
```

### 8.5 First login

Browse to `https://webmail.example.com` and log in with a full email address and its password. You can log in as either `john@example.com` or just `john`.

---

## 9. The `addmail` Script

Creating a mailbox by hand means editing two files, generating a password hash, fixing permissions, and reloading two services. The `addmail` script does all of it in one go.

### 9.1 Create the script

Create `/usr/local/bin/addmail`:

```bash
#!/bin/bash
# addmail — create a virtual mailbox, set its password, reload services
# Usage: sudo addmail name@example.com

set -euo pipefail

DOMAIN="example.com"
VMAILBOX="/etc/postfix/vmailbox"
USERS="/etc/dovecot/users"
MAILROOT="/var/mail/vhosts/${DOMAIN}"

if [[ $# -ne 1 ]]; then
    echo "Usage: sudo addmail name@${DOMAIN}"
    exit 1
fi

ADDRESS="$1"

# Validate the address belongs to our domain
if [[ "${ADDRESS}" != *"@${DOMAIN}" ]]; then
    echo "Error: address must end in @${DOMAIN}"
    exit 1
fi

LOCALPART="${ADDRESS%%@*}"

if [[ -z "${LOCALPART}" ]]; then
    echo "Error: local part cannot be empty"
    exit 1
fi

# Refuse if the address already exists
if grep -q "^${ADDRESS}:" "${USERS}" 2>/dev/null; then
    echo "Error: ${ADDRESS} already exists in ${USERS}"
    exit 1
fi

# Prompt for the password (twice)
read -rsp "Enter password for ${ADDRESS}: " PASSWORD
echo
read -rsp "Confirm password: " PASSWORD_CONFIRM
echo

if [[ "${PASSWORD}" != "${PASSWORD_CONFIRM}" ]]; then
    echo "Error: passwords do not match"
    exit 1
fi

if [[ ${#PASSWORD} -lt 8 ]]; then
    echo "Error: password must be at least 8 characters"
    exit 1
fi

# 1. Add the mailbox entry
echo "${ADDRESS}    ${DOMAIN}/${LOCALPART}/" | sudo tee -a "${VMAILBOX}" > /dev/null
sudo postmap "${VMAILBOX}"

# 2. Generate the password hash and add the user
HASH=$(doveadm pw -s SHA512-CRYPT -p "${PASSWORD}")
echo "${ADDRESS}:${HASH}" | sudo tee -a "${USERS}" > /dev/null
sudo chown root:dovecot "${USERS}"
sudo chmod 640 "${USERS}"

# 3. Create the mail directory
sudo mkdir -p "${MAILROOT}/${LOCALPART}"
sudo chown -R vmail:vmail "${MAILROOT}"
sudo chmod -R 770 "${MAILROOT}"

# 4. Reload services
sudo systemctl reload postfix
sudo systemctl reload dovecot

echo "Mailbox ${ADDRESS} created successfully."
echo "You can now log in to Roundcube as ${ADDRESS} or ${LOCALPART}."
```

### 9.2 Make it executable

```bash
sudo chmod +x /usr/local/bin/addmail
```

### 9.3 Usage

```bash
sudo addmail jane@example.com
```

The script will:

1. Prompt for a password twice (minimum 8 characters).
2. Append the address to `/etc/postfix/vmailbox` and run `postmap`.
3. Generate a SHA512-CRYPT hash and append it to `/etc/dovecot/users` (with `root:dovecot` ownership and mode `640`).
4. Create `/var/mail/vhosts/example.com/jane/` owned by `vmail:vmail`.
5. Reload Postfix and Dovecot.

The mailbox works immediately: Dovecot creates the folder structure on first delivery, and you can log in to Roundcube as `jane` or `jane@example.com`.

### 9.4 Verify

```bash
sudo cat /etc/postfix/vmailbox          # new address should be listed
sudo cat /etc/dovecot/users             # shows the address and a password hash
```

Then email the new address from Gmail and watch for it:

```bash
sudo journalctl -u mail-inbound -n 3 --no-pager
sudo ls -t /var/mail/vhosts/example.com/jane/new/ | head -3
```

---

## 10. Cloudflare Email Routing & Worker

### 10.1 Set up the catch-all rule

In the Cloudflare dashboard:

1. Go to **Email → Email Routing**.
2. Click **Add destination address** and enter `john@example.com` (your real mailbox — this is where catch-all mail lands if no specific rule matches).
3. Go to **Email Routing → Catch-all address** and click **Edit**.
4. Set the action to **Send to a Worker** and select your Worker (created below).
5. Enable the catch-all.

### 10.2 Create the Worker

Create a new Worker (e.g. `mail-inbound`) with the following code:

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    // Optional: verify a shared secret
    const auth = request.headers.get('Authorization');
    if (auth !== `Bearer ${env.INBOUND_SECRET}`) {
      return new Response('Unauthorized', { status: 401 });
    }

    const raw = await request.text();

    // Forward to your server's inbound endpoint
    const response = await fetch('https://mail.example.com/inbound', {
      method: 'POST',
      headers: { 'Content-Type': 'message/rfc822' },
      body: raw,
    });

    return new Response(await response.text(), {
      status: response.status,
    });
  },
};
```

### 10.3 Add the secret

In the Worker's **Settings → Variables**, add:

| Variable name | Type | Value |
|---|---|---|
| `INBOUND_SECRET` | Secret | A long random string |

### 10.4 The inbound endpoint on your server

Your server needs an HTTPS endpoint that accepts the raw message and hands it to Postfix. A minimal implementation using a small Node.js or Python service:

**Python (Flask) example** — save as `/opt/mail-inbound/server.py`:

```python
from flask import Flask, request
import smtplib
from email import message_from_string

app = Flask(__name__)

INBOUND_SECRET = "REPLACE_WITH_LONG_RANDOM_STRING"

@app.route('/inbound', methods=['POST'])
def inbound():
    auth = request.headers.get('Authorization', '')
    if auth != f'Bearer {INBOUND_SECRET}':
        return 'Unauthorized', 401

    raw = request.get_data(as_text=True)
    msg = message_from_string(raw)

    # Hand the message to Postfix on localhost
    with smtplib.SMTP('127.0.0.1', 25) as smtp:
        smtp.sendmail(msg['From'], msg['To'], raw)

    return 'OK', 200

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8080)
```

Run it as a systemd service (`/etc/systemd/system/mail-inbound.service`):

```ini
[Unit]
Description=Mail inbound endpoint
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/mail-inbound/server.py
Restart=always
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now mail-inbound
```

Put it behind Nginx or Apache with TLS on `mail.example.com`, or use Cloudflare Tunnel to avoid opening port 443 directly.

### 10.5 Why no Cloudflare DNS change is needed for new mailboxes

The catch-all rule sends **every** address at your domain to the Worker, and your server decides whether the mailbox exists. A message for an address that isn't in `/etc/postfix/vmailbox` is rejected at your end. This means you can create new mailboxes at any time without touching DNS.

---

## 11. Brevo — Outbound Relay

### 11.1 Get your SMTP credentials

1. Log in to Brevo.
2. Go to **SMTP & API → SMTP**.
3. Note the SMTP server (`smtp-relay.brevo.com`), port (`587`), and your SMTP login/password.
4. If you haven't created an SMTP key yet, generate one.

### 11.2 Configure Postfix to relay through Brevo

This is already covered in [Section 6.5](#65-create-the-brevo-sasl-password-file). The key settings in `/etc/postfix/main.cf`:

```ini
relayhost = [smtp-relay.brevo.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
```

### 11.3 Authenticate your domain in Brevo

1. Go to **Senders, Domains & Dedicated IPs → Domains**.
2. Click **Add a domain** and enter `example.com`.
3. Brevo shows the DNS records to add (SPF, DKIM, and optionally a "Brevo code" verification record).
4. Add each record in Cloudflare DNS exactly as shown.
5. Click **Verify** in Brevo. Wait for DNS propagation, then re-verify.

### 11.4 Turn off tracking (recommended for deliverability)

In the Brevo dashboard, open **SMTP & API → Settings** and look for the options for tracking opens and clicks. Turn them off if available. Menu names change, so search the settings for "tracking".

> **Why:** Brevo's tracking injects a hidden `<img>` pixel into HTML mail and rewrites links. The pixel makes ordinary mail look like a newsletter to spam filters, and Brevo also adds `List-Unsubscribe` headers for the same reason. Plain-text mail can't contain the pixel, which is one more reason to compose in plain text (see [Section 16](#16-fixing-the-display-name-from-header)).

---

## 12. Webmin — Server Management

### 12.1 Install

```bash
wget -qO - https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh | sudo bash
sudo apt install -y webmin
```

Or use the official install script from [webmin.com](https://webmin.com).

Webmin listens on port **10000** by default. Open it in the firewall:

```bash
sudo ufw allow 10000/tcp
```

Browse to `https://your-server-ip:10000` and log in with your root or sudo user credentials.

### 12.2 Viewing users in Webmin

It depends on which users you mean. Your mail accounts aren't regular Linux users, so they don't appear in the usual Webmin user list.

#### Linux system users (such as `ubuntu`)

Go to **System → Users and Groups** in Webmin. This lists real system accounts. The mailboxes you created with `addmail` won't be here, because they're virtual users stored in a file, and `vmail` is the only related system account you'll see.

#### Mail users (`john@example.com` and so on)

Webmin has no built-in screen for this kind of Dovecot virtual user, so view them as files. Menu names vary slightly by Webmin version, so look for the closest match:

- **Tools → File Manager** (or **Others → File Manager** in older versions): open `/etc/dovecot/users` for the logins and `/etc/postfix/vmailbox` for the mailbox list. The users file contains password hashes, so don't share screenshots of it.
- **Tools → Command Shell**: run `cat /etc/postfix/vmailbox` (Webmin runs as root, so you may not need `sudo`).
- **Servers → Dovecot IMAP/POP3 Server → Edit Config Files**: this lists Dovecot's config files, but may not include the users file. The File Manager is more reliable.

To see just the addresses without the hashes, in **Tools → Command Shell** run:

```bash
cut -d: -f1 /etc/dovecot/users
```

#### Seeing which mailboxes exist on disk

Open **Tools → File Manager** and browse to `/var/mail/vhosts/example.com/`. There's one folder per mailbox that has received or stored mail.

### 12.3 Adding or removing users in Webmin

Keep using the `addmail` script over SSH, or from **Tools → Command Shell** in Webmin. Editing the two files by hand in Webmin also works, but you need to run `postmap /etc/postfix/vmailbox` afterwards and keep the `/etc/dovecot/users` permissions as `root:dovecot` with mode `640`.

---

## 13. Managing Mail Users

### 13.1 Create a mailbox

```bash
sudo addmail jane@example.com
```

See [Section 9](#9-the-addmail-script) for full details.

### 13.2 Change a password

Generate a new hash:

```bash
doveadm pw -s SHA512-CRYPT
```

You'll be prompted for the password twice. Then edit the users file:

```bash
sudo nano /etc/dovecot/users
```

Replace the hash after the colon for the address in question. No service reload is needed — Dovecot reads the file on each authentication.

### 13.3 Remove a mailbox

1. Delete the address's line from `/etc/dovecot/users`.
2. Delete the address's line from `/etc/postfix/vmailbox`.
3. Rebuild the map and reload:

```bash
sudo postmap /etc/postfix/vmailbox
sudo systemctl reload postfix dovecot
```

The mail files stay in `/var/mail/vhosts/example.com/<name>/` until you delete them yourself, so back them up first if you might need them:

```bash
sudo tar -czf /root/mail-backup-$(date +%F).tar.gz /var/mail/vhosts/example.com/
```

---

## 14. Aliases — Extra Addresses into an Existing Mailbox

If you just want `info@` or `admin@` to land in the `john` mailbox, you don't need a new mailbox. Create an alias map:

```bash
echo "info@example.com john@example.com" | sudo tee -a /etc/postfix/virtual_aliases
sudo postconf -e "virtual_alias_maps = hash:/etc/postfix/virtual_aliases"
sudo postmap /etc/postfix/virtual_aliases
sudo systemctl reload postfix
```

Add more aliases by appending lines to that file and running `sudo postmap /etc/postfix/virtual_aliases` again. If you've already set `virtual_alias_maps` once, skip the `postconf` line on later additions.

### 14.1 Required alias: postmaster

Your DMARC record asks for reports at `postmaster@example.com`, which needs to exist. Create an alias into your mailbox if you haven't yet:

```bash
echo "postmaster@example.com john@example.com" | sudo tee -a /etc/postfix/virtual_aliases
sudo postconf -e "virtual_alias_maps = hash:/etc/postfix/virtual_aliases"
sudo postmap /etc/postfix/virtual_aliases
sudo systemctl reload postfix
```

### 14.2 Other useful aliases

```bash
cat <<'EOF' | sudo tee -a /etc/postfix/virtual_aliases
admin@example.com       john@example.com
webmaster@example.com   john@example.com
abuse@example.com       john@example.com
EOF
sudo postmap /etc/postfix/virtual_aliases
sudo systemctl reload postfix
```

---

## 15. Sending as a Different Address

### 15.1 From Roundcube

Log in as the new user, or add it as an identity under **Settings → Identities** while logged in as `john`.

Each mailbox has its own identities, so repeat this for every account you add. You can also create more identities under the same login if you want to send as several names or addresses.

### 15.2 From a mail client

Use the new full address as the SMTP username and its password. For example, in Thunderbird:

- **Incoming:** IMAP, `mail.example.com`, port 993, SSL/TLS, password = the mailbox password.
- **Outgoing:** SMTP, `mail.example.com`, port 587, STARTTLS, username = `jane@example.com`, password = the mailbox password.

### 15.3 From Brevo

Any address at the verified domain can normally send through Brevo, but if Brevo rejects one, check that its sender settings allow the whole domain.

---

## 16. Fixing the Display Name (From Header)

### 16.1 The problem

When you send mail, recipients see a display name pulled from the `From:` header. If no display name is set, the name falls back to the Linux account's full name — which on a fresh VPS is something like `ubuntu`. Headers may show:

```text
From: <ubuntu@example.com>
```

or

```text
From: <hacker@example.com>
```

There's no display name, and the address itself may be unprofessional. Words like "hacker", plus a subject of "test" and a one-word body, are exactly what spam filters treat as suspicious.

### 16.2 The fix (Roundcube)

1. In Roundcube, go to **Settings → Identities**.
2. Click your identity (`john@example.com`).
3. Set **Display Name** to what you want recipients to see, for example `John Doe`.
4. Optionally set **Organization** and a **Signature**.
5. **Save**, then send a fresh test to Gmail.

Gmail may show an old cached name for a while on existing conversations. A brand-new message to a different Gmail address is the cleanest test.

### 16.3 The fix (command line)

Mail sent with `sendmail` or `mail` from the command line uses the Linux account's full name. To change it:

```bash
sudo chfn -f "John Doe" ubuntu
```

Then send again. For mail from scripts or cron, you can also set the name explicitly by adding a header:

```bash
echo "Subject: Test" | sendmail -F "John Doe" -f john@example.com recipient@gmail.com
```

### 16.4 Check what's actually being sent

In Gmail, open the message, click the three dots, then **Show original**, and look at the `From:` line. It shows the exact display name and address your server sent, so you can tell whether it was changed at the source.

If the `From:` line still says `ubuntu` after you've set the Roundcube identity, check exactly how you sent it (Roundcube or terminal) and compare again.

### 16.5 Send plain text instead of HTML

The message may go out as HTML-only with Brevo's tracking added. In Roundcube, go to **Settings → Preferences → Composing Messages** and set **Compose HTML messages** to **never**. Plain-text mail gets sent as plain text, and Brevo can't inject a hidden tracking image into it.

---

## 17. Email Deliverability — SPF, DKIM, DMARC

Since you relay through Brevo, your home IP doesn't matter, so deliverability comes down to authentication, domain reputation, and message content. Work through these in order.

### 17.1 Read the verdict Gmail gives you

Send a message to Gmail, open it, click the three dots, then **Show original**. At the top, Gmail shows:

```text
SPF:   PASS
DKIM:  PASS  with domain example.com
DMARC: PASS
```

| Result | What to fix |
|---|---|
| SPF FAIL/SOFTFAIL | Step 17.2 |
| DKIM FAIL, or PASS but with domain `brevo.com` | Step 17.3 |
| DMARC FAIL | Fix SPF or DKIM, since DMARC needs one of them to pass and match your From domain |
| All PASS but still spam | Steps 17.4 and 17.5 (reputation and content) |

### 17.2 SPF (one record only)

In Cloudflare → DNS, there must be exactly one TXT record starting with `v=spf1`. Merge Cloudflare Email Routing and Brevo into it:

```text
v=spf1 include:_spf.mx.cloudflare.net include:spf.brevo.com ~all
```

Use the include value from Brevo's domain-setup page if it differs. Two SPF records cause a permanent error.

### 17.3 DKIM and domain authentication in Brevo

In Brevo, open **Senders, Domains & Dedicated IPs → Domains**, select `example.com` and use **Authenticate**. It should show the DKIM and Brevo code records as verified, and (if offered) DMARC as well. Add any missing records in Cloudflare exactly as shown, and set them to **DNS only** (grey cloud) if they're CNAMEs.

Brevo must sign with your domain. If "Show original" says the DKIM domain is `brevo.com`, authentication isn't finished.

### 17.4 DMARC

Keep your record, and make sure someone can receive the reports:

```text
v=DMARC1; p=none; rua=mailto:postmaster@example.com
```

`postmaster@` doesn't exist on your server yet by default, so create an alias to your mailbox (see [Section 14.1](#141-required-alias-postmaster)). Move to `p=quarantine` after a couple of weeks of clean passes, since receivers trust stricter DMARC more.

### 17.5 Score it

Send one message to the address at [mail-tester.com](https://mail-tester.com). It lists exactly what's failing (SPF, DKIM, DMARC, blocklists, content) with a score out of 10. Aim for 9 or more.

---

## 18. Reputation & Content Best Practices

### 18.1 Reputation: the part that takes time

- **New domain with no history.** Mail from a fresh domain often goes to spam at first. Start with low volume to real people you know.
- **Engagement trains Gmail.** Open your test messages in spam, click **Report not spam**, reply from Gmail to your address, and add your address to the recipient's contacts. A few rounds usually moves you to the inbox.
- **TLD reputation.** Some TLDs (e.g. `.xyz`) have a worse reputation with filters than `.com` or `.co.za`, which is a real disadvantage, so be patient. A cheap `.co.za` or `.com` domain would help if this becomes a long-term address.
- **Send from the address you authenticated**, and keep the display name consistent (the Roundcube identity).

### 18.2 Message content

- Write a normal subject line and body, not just a link or a single word.
- Avoid URL shorteners, attachments on first contact, all-caps and heavy images.
- Plain text or simple HTML is fine. Roundcube's default is good.
- A short test like "hi" or "test" is more likely to be flagged than a real message.

### 18.3 What you can't control

The Brevo free tier uses shared sending IPs, so your reputation partly depends on other senders. If authentication all passes, mail-tester scores well and Gmail still spam-folders you, that shared-IP reputation and the new domain are the usual causes, and steady low-volume legitimate use fixes it over a few weeks.

---

## 19. Troubleshooting

### 19.1 Mail not arriving

```bash
# Check the inbound Worker logs
sudo journalctl -u mail-inbound -n 50 --no-pager

# Check Postfix logs
sudo tail -50 /var/log/mail.log

# Verify the mailbox exists
grep "jane@example.com" /etc/postfix/vmailbox

# Test delivery manually
echo "Test body" | sendmail -F "Test" -f john@example.com jane@example.com
```

### 19.2 Mail not sending

```bash
# Check Postfix can reach Brevo
telnet smtp-relay.brevo.com 587

# Check the SASL password file
sudo postmap -q "[smtp-relay.brevo.com]:587" hash:/etc/postfix/sasl_passwd

# Check for relay errors
sudo tail -50 /var/log/mail.log | grep -i "relay\|sasl\|auth"
```

### 19.3 Authentication failures in Roundcube

```bash
# Verify the password hash works
doveadm pw -s SHA512-CRYPT -t '{SHA512-CRYPT}hash-from-users-file' -p "the-password"

# Check Dovecot logs
sudo journalctl -u dovecot -n 50 --no-pager

# Verify file permissions
ls -la /etc/dovecot/users    # should be root:dovecot, mode 640
```

### 19.4 "User unknown" rejections

The address isn't in `/etc/postfix/vmailbox`. Either create it with `addmail` or check for typos:

```bash
sudo postmap -q jane@example.com hash:/etc/postfix/vmailbox
```

### 19.5 Stale hostname in headers

If you see a placeholder hostname (e.g. `mail.example.co.za`) in `Received:` headers:

```bash
grep -rn "example.co.za" /etc/hosts /etc/mailname /etc/postfix /etc/dovecot /etc/roundcube 2>/dev/null
sudo sed -i 's/example\.co\.za/example.com/g' /etc/hosts
getent hosts 192.168.1.100          # should say mail.example.com
sudo systemctl restart postfix
```

The `grep` lists any other files that still contain the placeholder domain.

---

## 20. Maintenance & Operations

### 20.1 Daily

- Nothing. The system runs itself.

### 20.2 Weekly

```bash
# Check disk space (mail accumulates)
df -h /var/mail

# Check for failed services
systemctl --failed

# Review mail logs for anomalies
sudo grep -i "error\|fatal\|reject" /var/log/mail.log | tail -20
```

### 20.3 Monthly

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Renew Let's Encrypt certificates (certbot does this automatically, but verify)
sudo certbot renew --dry-run

# Review mailbox sizes
sudo du -sh /var/mail/vhosts/example.com/* | sort -h

# Check DMARC reports (if you have a parser)
# Consider parsing rua reports with dmarcian, dmarcSieve, or a simple Python script
```

### 20.4 Backups

```bash
# Back up mail data
sudo tar -czf /root/mail-vhosts-$(date +%F).tar.gz /var/mail/vhosts/

# Back up configuration
sudo tar -czf /root/mail-config-$(date +%F).tar.gz \
    /etc/postfix/ /etc/dovecot/ /etc/roundcube/ /etc/hosts /etc/mailname

# Copy off-server
scp /root/mail-*-$(date +%F).tar.gz backup-user@backup-server:/backups/
```

### 20.5 Log rotation

Postfix and Dovecot logs are handled by `rsyslog` and `logrotate` by default. Verify:

```bash
ls /etc/logrotate.d/postfix /etc/logrotate.d/dovecot
```

---

## 21. Security Notes

- **Never share `/etc/dovecot/users`** — it contains password hashes. If leaked, every mailbox is compromised.
- **Keep `/etc/postfix/sasl_passwd` at mode 600** — it contains your Brevo SMTP credentials in plain text.
- **Use strong, unique passwords** for each mailbox. The `addmail` script enforces a minimum of 8 characters; consider 16+.
- **Enable 2FA on Brevo** — if your Brevo account is compromised, an attacker can send mail as your domain.
- **Keep Webmin on a non-standard port** or restrict it by IP in the firewall. Port 10000 is constantly scanned by bots.
- **Fail2ban** — install it to block brute-force attempts on SSH and Postfix:

  ```bash
  sudo apt install -y fail2ban
  sudo systemctl enable --now fail2ban
  ```

- **TLS everywhere** — IMAPS on 993, submission on 587 with STARTTLS, HTTPS for Roundcube and Webmin. Never allow plain-text authentication over the network.
- **Regular updates** — `unattended-upgrades` is your friend:

  ```bash
  sudo apt install -y unattended-upgrades
  sudo dpkg-reconfigure -plow unattended-upgrades
  ```

---

## 22. Appendix — Full Configuration Files

### 22.1 `/etc/postfix/main.cf`

See [Section 6.2](#62-main-configuration) for the complete annotated configuration.

### 22.2 `/etc/postfix/master.cf` (submission section)

See [Section 6.3](#63-master-configuration).

### 22.3 `/etc/postfix/vmailbox`

```text
john@example.com    example.com/john/
jane@example.com    example.com/jane/
```

### 22.4 `/etc/postfix/virtual_aliases`

```text
info@example.com        john@example.com
admin@example.com       john@example.com
webmaster@example.com   john@example.com
abuse@example.com       john@example.com
postmaster@example.com  john@example.com
```

### 22.5 `/etc/postfix/sasl_passwd`

```text
[smtp-relay.brevo.com]:587    your-brevo-smtp-login:your-brevo-smtp-key
```

### 22.6 `/etc/dovecot/users`

```text
john@example.com:{SHA512-CRYPT}$6$rounds=50000$REPLACE_WITH_REAL_SALT$REPLACE_WITH_REAL_HASH
jane@example.com:{SHA512-CRYPT}$6$rounds=50000$REPLACE_WITH_REAL_SALT$REPLACE_WITH_REAL_HASH
```

### 22.7 `/etc/dovecot/dovecot.conf`

```ini
protocols = imap lmtp
listen = *, ::
```

### 22.8 `/etc/dovecot/conf.d/10-mail.conf` (key lines)

```ini
mail_location = maildir:/var/mail/vhosts/%d/%n
mail_uid = 5000
mail_gid = 5000
mail_privileged_group = vmail
```

### 22.9 `/etc/dovecot/conf.d/10-auth.conf` (key lines)

```ini
disable_plaintext_auth = yes
auth_mechanisms = login plain
!include auth-passwdfile.conf.ext
```

### 22.10 `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`

```ini
passdb {
  driver = passwd-file
  args = scheme=SHA512-CRYPT /etc/dovecot/users
}

userdb {
  driver = static
  args = uid=5000 gid=5000 home=/var/mail/vhosts/%d/%n
}
```

### 22.11 `/etc/dovecot/conf.d/10-master.conf` (key sockets)

```ini
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

### 22.12 `/etc/hosts`

```text
127.0.0.1       localhost
203.0.113.10    mail.example.com mail
192.168.1.100   mail.example.com mail
```

### 22.13 `/etc/mailname`

```text
example.com
```

### 22.14 Cloudflare DNS summary

| Type | Name | Content | Proxy |
|---|---|---|---|
| A | `mail` | `203.0.113.10` | DNS only |
| A | `webmail` | `203.0.113.10` | DNS only |
| MX | `@` | `route1.mx.cloudflare.net` | — |
| MX | `@` | `route2.mx.cloudflare.net` | — |
| MX | `@` | `route3.mx.cloudflare.net` | — |
| TXT | `@` | `v=spf1 include:_spf.mx.cloudflare.net include:spf.brevo.com ~all` | — |
| TXT | `brevo2._domainkey` | `v=DKIM1; k=rsa; p=...` | DNS only |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:postmaster@example.com` | — |

### 22.15 Directory structure

```text
/etc/
├── dovecot/
│   ├── dovecot.conf
│   ├── users                          # virtual user passwords (root:dovecot, 640)
│   └── conf.d/
│       ├── 10-auth.conf
│       ├── 10-mail.conf
│       ├── 10-master.conf
│       └── auth-passwdfile.conf.ext
├── postfix/
│   ├── main.cf
│   ├── master.cf
│   ├── vmailbox                       # mailbox list (plain text)
│   ├── vmailbox.db                    # compiled hash
│   ├── virtual_aliases                # alias list (plain text)
│   ├── virtual_aliases.db             # compiled hash
│   ├── sasl_passwd                    # Brevo credentials (root, 600)
│   └── sasl_passwd.db                 # compiled hash
├── roundcube/
│   └── config.inc.php
├── hosts
└── mailname

/var/mail/vhosts/example.com/
├── john/
│   ├── cur/
│   ├── new/
│   ├── tmp/
│   └── Maildir
└── jane/
    ├── cur/
    ├── new/
    └── tmp/

/usr/local/bin/
└── addmail                            # mailbox creation script

/opt/mail-inbound/
└── server.py                          # inbound HTTP endpoint
```

---

## Document History

| Date | Change |
|---|---|
| 2026-09-30 | Initial complete documentation, redacted for public sharing |

---

*This document is provided as-is for educational purposes. Self-hosted email requires ongoing maintenance and vigilance. Always test changes in a staging environment before applying them to production.*
