# Client Setup Guide for mTLS



## Prerequisites

- OpenSSL installed (1.1.1+)
- CA certificate (`example-rootCA.crt`) from your server administrator
- A web browser (Firefox, Chrome, Safari, or Edge)

---

## Step 1: Generate Client Keypair
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out client.key
```

**Why 3072 bits?** Stronger security than 2048, recommended for long-lived certificates.

---

## Step 2: Create CSR Configuration (Sample CSR Config Template)

Save this as `csr.conf`:
```ini
[req]
default_bits = 3072
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = Your Name                    # CUSTOMIZE
O = Your Organization             # CUSTOMIZE
C = Country                           # CUSTOMIZE
ST = State                       # OPTIONAL
L = City                         # OPTIONAL

[v3_req]
subjectAltName = DNS:localhost, DNS:*.example.com
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth    # REQUIRED for client auth
```

**Critical:** The `extendedKeyUsage = clientAuth` is required. Without it, browsers will gray out your certificate during selection.

---

## Step 3: Generate Certificate Signing Request
```bash
openssl req -new -key client.key -out client.csr -config csr.conf
```

**Verify CSR includes SAN:**
```bash
openssl req -in client.csr -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Step 4: Sign the Certificate

**Option A: Self-signed with demo CA (testing)**
```bash
openssl x509 -req -in client.csr \
    -CA example-rootCA.crt -CAkey demo_root.key \
    -CAcreateserial -out client.crt -days 365 -sha256 \
    -extfile csr.conf -extensions v3_req
```

**Option B: Submit to organizational CA**
- Upload `client.csr` to your CA portal
- Download the issued `client.crt`

---

## Step 5: Create PKCS#12 Bundle
```bash
openssl pkcs12 -export -out client.p12 \
    -inkey client.key \
    -in client.crt \
    -certfile example-rootCA.crt \
    -passout pass:YourStrongPassword123
```

---

## Step 6: Import into Browser/OS

### macOS (Keychain)
```bash
# Import via GUI
open client.p12

# Or command line
security import client.p12 -k ~/Library/Keychains/login.keychain-db
```

**Verify import:**
```bash
security find-identity -v -p sslclient
```

### Windows (Certificate Manager)

1. Double-click `client.p12`
2. Select "Current User" → Next
3. Enter password from Step 5 → Next
4. "Automatically select certificate store" → Finish

**Verify import:**
```powershell
Get-ChildItem -Path Cert:\CurrentUser\My
```

### Linux (Firefox)

1. Firefox → Settings → Privacy & Security
2. Certificates → View Certificates
3. Your Certificates → Import
4. Select `client.p12`, enter password

**Verify import:**
```bash
certutil -L -d ~/.mozilla/firefox/YOUR_PROFILE
```

### Chrome/Chromium

Chrome uses the OS certificate store:
- **macOS/Windows:** Follow OS instructions above
- **Linux:** Import via system certificate manager or use `pk12util`

**Important:** Restart your browser completely after import.

---

## Step 7: Verify Setup

### Command Line Test
```bash
# Without certificate (should fail with 403)
curl -vk https://example.test/

# With certificate (should succeed)
curl -vk --cert client.p12:YourStrongPassword123 https://example.test/

# Alternative with separate cert/key
curl -vk --cert client.crt --key client.key https://example.test/
```

### Browser Test

1. Navigate to `https://example.test/`
2. Browser should prompt for certificate selection (first visit only)
3. Select your client certificate
4. Page should load successfully (HTTP 200)

**If no prompt appears:**
- Certificate not imported to correct store → Re-import
- Browser cached decision → Clear cache, restart browser
- Certificate missing `clientAuth` extension → Regenerate CSR

---

## Troubleshooting Quick Reference

| Issue | Cause | Solution |
|-------|-------|----------|
| Browser doesn't show certificate dialog | Not imported or browser not restarted | Re-import, fully restart browser |
| Certificate grayed out in selection | Missing `extendedKeyUsage = clientAuth` | Regenerate CSR with correct extension |
| curl: "MAC verification failed" | Wrong PKCS#12 password | Re-export with correct password |
| "No certificate provided" in logs | Certificate not sent | Check browser store, restart browser |
| "Certificate verify failed" | Wrong CA or expired cert | Verify CA matches server, check expiry |

**For detailed error diagnostics, see [0_troubleshooting_guide.pdf](0_troubleshooting_guide.pdf)**

---

## Verification Commands
```bash
# Check certificate details
openssl x509 -in client.crt -noout -text

# Verify certificate chain
openssl verify -CAfile example-rootCA.crt client.crt

# Check PKCS#12 integrity
openssl pkcs12 -in client.p12 -noout -info -passin pass:YourPassword

# Test TLS handshake
openssl s_client -connect example.test:443 \
    -cert client.crt -key client.key \
    -CAfile example-rootCA.crt
```

---

## Multi-Device Setup

To use the same certificate on multiple devices:

1. **Transfer PKCS#12 file securely**
2. **Import on each device** following OS-specific instructions above
3. **Use same password** you set during export in Step 5

**Security note:** Each device import uses the same private key. For higher security, generate separate certificates per device.

---

## Complete Automation Script
```bash
#!/bin/bash
# Automated client certificate setup

set -e  # Exit on error

# Configuration
KEYFILE="client.key"
CSRFILE="client.csr"
CRTFILE="client.crt"
P12FILE="client.p12"
CAFILE="example-rootCA.crt"
CAKEY="demo_root.key"
PASSWORD="YourStrongPassword123"

echo " Step 1: Generating keypair..."
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out "$KEYFILE"

echo " Step 2: Creating CSR..."
openssl req -new -key "$KEYFILE" -out "$CSRFILE" -config csr.conf

echo " Step 3: Signing certificate..."
openssl x509 -req -in "$CSRFILE" \
    -CA "$CAFILE" -CAkey "$CAKEY" \
    -CAcreateserial -out "$CRTFILE" -days 365 -sha256 \
    -extfile csr.conf -extensions v3_req

echo " Step 4: Creating PKCS#12 bundle..."
openssl pkcs12 -export -out "$P12FILE" \
    -inkey "$KEYFILE" -in "$CRTFILE" -certfile "$CAFILE" \
    -passout "pass:${PASSWORD}"

echo ""
echo " Setup complete!"
echo " Import $P12FILE into your browser/OS"
echo "   Password: $PASSWORD"
echo " Test: curl --cert $P12FILE:$PASSWORD https://example.test/"
```

---