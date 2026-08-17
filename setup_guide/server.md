# Server Setup Reference for mTLS

Quick reference for configuring NGINX to enforce mutual TLS authentication.

---

## NGINX vhost snippet
```nginx
ssl_verify_client on;
ssl_verify_depth 1;
ssl_client_certificate /etc/nginx/certs/example-rootCA.crt;

location @ssl_verification_error {
    return 403 "Client certificate verification failed.";
}
```

## NGINX location snippet
```nginx
#proxy_set_header X-SSL-CLIENT-CERT $ssl_client_escaped_cert;
proxy_set_header X-SSL-CLIENT-CERT-SERIAL $ssl_client_serial;
proxy_set_header X-SSL-CLIENT-CERT-DN $ssl_client_s_dn;

error_page 495 =200 @ssl_verification_error;
```

## Demo Root CA commands
```bash
openssl genrsa -out demo_root.key 4096
openssl req -x509 -new -key demo_root.key -days 3650 \
    -subj "/CN=Artifact Demo Root CA" \
    -out example-rootCA.crt
```

## Demo Root CA summary
```
subject=CN = Artifact Demo Root CA
issuer=CN = Artifact Demo Root CA
notBefore=Aug 20 15:56:01 2025 GMT
notAfter=Aug 18 15:56:01 2035 GMT
SHA1 Fingerprint=3E:64:33:FA:A0:BF:D8:F1:B3:58:59:A4:05:33:EB:4A:D7:E9:B2:D6
```

## docker-compose example
```yaml
services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy:1.7
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config-templates/nginx-proxy/vhost.d:/etc/nginx/vhost.d:ro
      - ./config-templates/nginx-proxy/certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro

  app:
    image: ghcr.io/example/mtls-demo:latest
    environment:
      - VIRTUAL_HOST=example.test
      - VIRTUAL_PORT=5000
```

## Client certificate for testing
```bash
# Generate a client key and CSR
openssl genrsa -out client.key 4096
openssl req -new -key client.key -subj "/CN=testuser" -out client.csr

# Create minimal extensions for client auth
printf "basicConstraints=CA:FALSE\nkeyUsage=digitalSignature\nextendedKeyUsage=clientAuth\n" > client_ext.cnf

# Sign the client certificate with the demo root
openssl x509 -req -in client.csr -CA example-rootCA.crt -CAkey demo_root.key \
    -CAcreateserial -out client.crt -days 365 -sha256 -extfile client_ext.cnf
```

## Quick verification
```bash
# Without a client cert you should see 403 from the proxy
curl -vk https://example.test/

# With the client cert you should reach the app
curl -vk --cert client.crt --key client.key https://example.test/
```

---

**Notes**: The proxy enforces mutual TLS at the host level.