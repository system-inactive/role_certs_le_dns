# role_certs_le_dns

Ansible role for automated SSL/TLS certificate issuance via **Let's Encrypt** using **DNS-01 challenge** through Cloudflare DNS.

Supports single-domain, wildcard (`*.example.com`), and multi-domain (SAN) certificates.

## How it works

1. Generates an RSA private key for the Let's Encrypt account
2. Generates an RSA private key for the certificate
3. Creates a Certificate Signing Request (CSR)
4. Requests a DNS-01 challenge from Let's Encrypt (ACME v2)
5. Creates the required TXT record in Cloudflare DNS
6. Waits for DNS propagation
7. Validates the challenge and retrieves the certificate
8. Cleans up temporary files (account key, CSR)
9. Sets ownership and permissions on the certificate directory

## Requirements

- Ansible 2.9+
- Collections:
  - `community.crypto`
  - `community.general`

Install collections:
```bash
ansible-galaxy collection install community.crypto community.general
```

- A Cloudflare account with API access to the domain's DNS zone
- The domain must be managed by Cloudflare

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `email_address` | `sample@example.com` | Email for Let's Encrypt account registration |
| `cloudflare_email_address` | `sample@example.com` | Cloudflare account email |
| `cloudflare_account_api_key` | `000...` | Cloudflare Global API Key |
| `CN` | `example.com` | Certificate Common Name (primary domain) |
| `is_wildcard` | `false` | Set to `true` to issue a wildcard certificate (`*.CN`) |
| `cert_path` | `./crt/{{ CN }}` | Directory to store the issued certificate files |
| `cert_name` | `cert.pem` | Filename for the certificate |
| `cert_chain` | `chain.pem` | Filename for the intermediate chain |
| `cert_fullchain` | `fullchain.pem` | Filename for the full chain (cert + intermediate) |
| `priv_key` | `priv.key` | Filename for the private key |
| `csr_file` | `req.csr` | Filename for the CSR (removed after issuance) |
| `le_key_path` | `/tmp` | Directory for the temporary Let's Encrypt account key |
| `le_key_filename` | `priv_le.key` | Filename for the temporary account key (removed after issuance) |
| `acme_directory` | `https://acme-staging-v02.api.letsencrypt.org/directory` | ACME directory URL. Use staging for tests, production URL for real certs |
| `subject_alt_name` | _(undefined)_ | List of SANs for multi-domain certificates (see example below) |

### ACME directory URLs

| Environment | URL |
|---|---|
| **Staging** (testing) | `https://acme-staging-v02.api.letsencrypt.org/directory` |
| **Production** (real certs) | `https://acme-v02.api.letsencrypt.org/directory` |

> **Note:** Always test with the staging URL first. Let's Encrypt production has strict rate limits.

## Example Playbook

### Single-domain certificate

```yaml
- hosts: localhost
  roles:
    - role: role_certs_le_dns
      vars:
        email_address: admin@example.com
        cloudflare_email_address: admin@example.com
        cloudflare_account_api_key: your_cloudflare_api_key
        CN: example.com
        cert_path: /etc/ssl/example.com
        acme_directory: https://acme-v02.api.letsencrypt.org/directory
```

### Wildcard certificate

```yaml
- hosts: localhost
  roles:
    - role: role_certs_le_dns
      vars:
        email_address: admin@example.com
        cloudflare_email_address: admin@example.com
        cloudflare_account_api_key: your_cloudflare_api_key
        CN: example.com
        is_wildcard: true
        cert_path: /etc/ssl/wildcard.example.com
        acme_directory: https://acme-v02.api.letsencrypt.org/directory
```

### Multi-domain certificate (SAN)

```yaml
- hosts: localhost
  roles:
    - role: role_certs_le_dns
      vars:
        email_address: admin@example.com
        cloudflare_email_address: admin@example.com
        cloudflare_account_api_key: your_cloudflare_api_key
        CN: example.com
        cert_path: /etc/ssl/example.com
        acme_directory: https://acme-v02.api.letsencrypt.org/directory
        subject_alt_name:
          - DNS:www.example.com
          - DNS:api.example.com
```

## Output files

After a successful run, the following files will be created in `cert_path`:

| File | Description |
|---|---|
| `cert.pem` | Domain certificate |
| `chain.pem` | Intermediate CA certificate |
| `fullchain.pem` | Full chain (certificate + intermediate), use this for most web servers |
| `priv.key` | Private key (permissions `0600`) |

The certificate directory is owned by `root:jenkins` with permissions `u=rwX,g=rX,o=rX`.

## Notes

- The role **removes the existing certificate directory** at `cert_path` before issuance. Make sure to back up existing certificates if needed.
- The Let's Encrypt account key and CSR are automatically deleted after successful issuance.
- DNS propagation wait time is 30 seconds. If your DNS changes propagate slowly, increase the `dns_propagation_wait` pause accordingly.

## License

MIT
