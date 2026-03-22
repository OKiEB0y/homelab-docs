# Wazuh Dashboard TLS — Fix "Not Secure" Browser Warning
**Date:** 2026-03-21
**Category:** security
**System:** Ubuntu 24.04 — [SIEM-HOST] ([SIEM-HOST-IP])
**Risk Level:** Low (no cert changes made; read-only audit + export only)

---

## Overview
The Wazuh dashboard at `https://[SIEM-HOST-IP]` uses a self-signed TLS certificate issued
by an internal Wazuh CA. Browsers show "Not Secure" because they do not trust this CA.
The certificate itself is properly formed — it already contains the required SAN for the
server's IP address (`IP Address:[SIEM-HOST-IP]`), so no certificate regeneration was needed.

The fix is to import the Wazuh root CA into the OS/browser trust store on each client
machine, after which the browser will silently trust the dashboard certificate.

---

## How the Certificate Was Created

The Wazuh installer (`wazuh-install.sh`) auto-generates all certificates using OpenSSL during installation. Here is exactly what it does — useful if you ever need to regenerate or replicate the process manually.

### Step 1 — Generate the Root CA

```bash
openssl req -x509 -new -nodes -newkey rsa:2048 \
  -keyout root-ca.key \
  -out root-ca.pem \
  -batch \
  -subj '/OU=Wazuh/O=Wazuh/L=California/' \
  -days 3650
```

This produces a self-signed root CA valid for 10 years with no password on the key (`-nodes`).

### Step 2 — Generate a Config File with SAN for the Dashboard

The installer templates an OpenSSL config (`dashboard.conf`) for each node:

```ini
[ req ]
prompt = no
default_bits = 2048
default_md = sha256
distinguished_name = req_distinguished_name
x509_extensions = v3_req

[req_distinguished_name]
C = US
L = California
O = Wazuh
OU = Wazuh
CN = dashboard

[ v3_req ]
authorityKeyIdentifier=keyid,issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = [SIEM-HOST-IP]
```

The `CN` and `IP.1` values are substituted per-node from `config.yml`.

### Step 3 — Generate the Dashboard Key Pair and CSR

```bash
openssl req -new -nodes -newkey rsa:2048 \
  -keyout dashboard-key.pem \
  -out dashboard.csr \
  -config dashboard.conf
```

### Step 4 — Sign the Certificate with the Root CA

```bash
openssl x509 -req \
  -in dashboard.csr \
  -CA root-ca.pem \
  -CAkey root-ca.key \
  -CAcreateserial \
  -out dashboard.pem \
  -extfile dashboard.conf \
  -extensions v3_req \
  -days 3650
```

The `-extensions v3_req` flag ensures the SAN (`IP.1 = [SIEM-HOST-IP]`) is embedded in the final cert — this is what browsers require.

### Final Cert Locations

| File | Path |
|------|------|
| Root CA cert | `/etc/wazuh-dashboard/certs/root-ca.pem` |
| Dashboard cert | `/etc/wazuh-dashboard/certs/dashboard.pem` |
| Dashboard key | `/etc/wazuh-dashboard/certs/dashboard-key.pem` |
| Indexer certs | `/etc/wazuh-indexer/certs/` |

### To Regenerate Manually (e.g. after IP change)

```bash
# 1. Create the config file with new IP
cat > /tmp/dashboard.conf << 'EOF'
[ req ]
prompt = no
default_bits = 2048
default_md = sha256
distinguished_name = req_distinguished_name
x509_extensions = v3_req

[req_distinguished_name]
C = US
L = California
O = Wazuh
OU = Wazuh
CN = dashboard

[ v3_req ]
authorityKeyIdentifier=keyid,issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = [SIEM-HOST-IP]
EOF

# 2. Generate new key and CSR
openssl req -new -nodes -newkey rsa:2048 \
  -keyout /tmp/dashboard-key.pem \
  -out /tmp/dashboard.csr \
  -config /tmp/dashboard.conf

# 3. Sign with existing Wazuh CA
sudo openssl x509 -req \
  -in /tmp/dashboard.csr \
  -CA /etc/wazuh-dashboard/certs/root-ca.pem \
  -CAkey /path/to/root-ca.key \
  -CAcreateserial \
  -out /tmp/dashboard.pem \
  -extfile /tmp/dashboard.conf \
  -extensions v3_req \
  -days 3650

# 4. Install and restart
sudo cp /tmp/dashboard.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo cp /tmp/dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo systemctl restart wazuh-dashboard
```

> **Note:** The root CA private key (`root-ca.key`) is only present during installation — the installer discards it after generating all certs. If you need to sign new certs, you must either re-run the Wazuh cert tool or generate a fresh CA.

---

## Findings — Certificate Audit

| Field | Value |
|-------|-------|
| Cert path | `/etc/wazuh-dashboard/certs/dashboard.pem` |
| Key path | `/etc/wazuh-dashboard/certs/dashboard-key.pem` |
| Root CA path | `/etc/wazuh-dashboard/certs/root-ca.pem` |
| Cert CN | `dashboard` |
| SAN | `IP Address:[SIEM-HOST-IP]` (present — browsers satisfied) |
| Valid from | 2026-03-21 |
| Valid until | 2036-03-18 (10-year cert) |
| Issuer | `OU=Wazuh, O=Wazuh, L=California` |
| CA fingerprint (SHA-256) | `[CA-FINGERPRINT]` |
| Chain verification | `dashboard.pem: OK` |

No cert issues found. No service restart was required.

---

## What Was Done

The Wazuh root CA certificate was exported to the `[admin-user]` home directory in two formats:

```bash
sudo cp /etc/wazuh-dashboard/certs/root-ca.pem /home/[admin-user]/wazuh-root-ca.pem
sudo cp /home/[admin-user]/wazuh-root-ca.pem /home/[admin-user]/wazuh-root-ca.crt
sudo chmod 644 /home/[admin-user]/wazuh-root-ca.*
sudo chown [admin-user]:[admin-user] /home/[admin-user]/wazuh-root-ca.*
```

- `wazuh-root-ca.pem` — standard PEM format; use for Firefox, Linux
- `wazuh-root-ca.crt` — identical content, `.crt` extension; Windows double-click install

---

## How to Download the CA File

From your Kali machine (or any machine with SSH access):

```bash
scp [admin-user]@[SIEM-HOST-IP]:/home/[admin-user]/wazuh-root-ca.crt .
```

Or on Windows PowerShell (if OpenSSH is installed):

```powershell
scp [admin-user]@[SIEM-HOST-IP]:/home/[admin-user]/wazuh-root-ca.crt C:\Users\YourName\Downloads\
```

---

## How to Import the CA Certificate

### Windows — Chrome / Edge (certmgr.msc method)

Chrome and Edge on Windows use the Windows OS certificate store.

1. Press `Win + R`, type `certmgr.msc`, press Enter
2. In the left pane, expand **Trusted Root Certification Authorities**
3. Right-click **Certificates** → **All Tasks** → **Import**
4. Click Next → Browse → select `wazuh-root-ca.crt`
5. Ensure "Place all certificates in the following store" shows **Trusted Root Certification Authorities**
6. Click Next → Finish → Yes (to the security prompt)
7. Close certmgr, restart Chrome/Edge, navigate to `https://[SIEM-HOST-IP]`
8. The padlock should now show as secure

**Verify the import:**
- In certmgr, under Trusted Root Certification Authorities → Certificates
- Look for an entry with Issued To/By: `Wazuh` and expiry `3/18/2036`

> Note: This trust applies to your Windows user profile only. For machine-wide trust
> (all users), use `certlm.msc` instead and run it as Administrator.

---

### Windows — Alternative: PowerShell (requires Administrator)

```powershell
Import-Certificate -FilePath "C:\Users\YourName\Downloads\wazuh-root-ca.crt" `
  -CertStoreLocation Cert:\LocalMachine\Root
```

---

### Firefox (all platforms)

Firefox maintains its own certificate store independent of the OS.

1. Open Firefox → hamburger menu → **Settings**
2. Search for "certificates" in the search bar, or navigate to **Privacy & Security**
3. Scroll to the bottom → click **View Certificates**
4. Click the **Authorities** tab → **Import**
5. Select `wazuh-root-ca.pem` (or `.crt` — both work)
6. Check **Trust this CA to identify websites**
7. Click OK → OK
8. Restart Firefox, navigate to `https://[SIEM-HOST-IP]`

---

### Linux — System-wide (Debian/Ubuntu/Kali)

```bash
sudo cp wazuh-root-ca.crt /usr/local/share/ca-certificates/wazuh-root-ca.crt
sudo update-ca-certificates
```

Chrome on Linux uses the system store after running `update-ca-certificates`.
Firefox on Linux still requires manual import via its own certificate manager (see above).

---

### macOS — Keychain

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain wazuh-root-ca.pem
```

Or via GUI: double-click the `.crt` file → Keychain Access opens → add to System keychain →
find the cert → Get Info → Trust → "When using this certificate" → **Always Trust**.

---

## Verification Steps

After import, verify the browser no longer shows a warning:

1. Navigate to `https://[SIEM-HOST-IP]`
2. Click the padlock icon — it should show "Connection is secure"
3. Click the padlock → "Certificate is valid" → confirm the issuer shows `Wazuh`

To verify the CA fingerprint matches before importing (recommended):

```bash
openssl x509 -in wazuh-root-ca.pem -fingerprint -sha256 -noout
```

Expected output:
```
sha256 Fingerprint=[CA-FINGERPRINT]
```

If the fingerprint does not match, do not import the file — re-download from the server.

---

## Rollback / Recovery

**To remove the imported CA from Windows:**
1. Open `certmgr.msc` → Trusted Root Certification Authorities → Certificates
2. Find the Wazuh cert → right-click → Delete

**To remove from Firefox:**
Settings → Privacy & Security → View Certificates → Authorities tab →
find Wazuh → Delete or Distrust

**Cert store on the server is unchanged** — no modifications were made to any Wazuh
certificate files. The only server-side action was copying the existing root CA to
`/home/[admin-user]/` for retrieval.

---

## Notes & Gotchas

- The cert is self-signed and browser trust is **client-side only**. Each machine/browser
  that needs to access the dashboard must import the CA individually.
- The cert expires **2036-03-18**. Set a calendar reminder to regenerate before expiry.
- The Wazuh CA (`root-ca.pem`) is not password protected. Treat it with care — anyone
  with this file could sign certs that your browser would trust. It is appropriate for
  internal use on a controlled home/lab network.
- Chrome on Windows requires a full browser restart (not just a tab reload) after
  importing the CA for the change to take effect.
- If Chrome still shows the warning after import, clear the HSTS cache:
  navigate to `chrome://net-internals/#hsts` → Domain Security Policy → delete `[SIEM-HOST-IP]`
