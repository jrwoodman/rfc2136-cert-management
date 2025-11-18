# RFC2136 Certificate Management

Automated SSL/TLS certificate management system using Let's Encrypt with RFC2136 DNS validation. This toolkit provides scripts for certificate retrieval, automated renewal, and deployment across multiple services and VMware ESXi hosts.

## Overview

This system automates the complete lifecycle of SSL/TLS certificates:
- **Certificate Acquisition**: Fetch wildcard certificates from Let's Encrypt using DNS-01 challenge via RFC2136
- **Format Conversion**: Automatically generate PKCS12 and JKS keystores for various applications
- **Repository Management**: Maintain certificates in a git-backed repository for version control
- **Automated Renewal**: Weekly certificate checks and updates via cron
- **Service Deployment**: Restart services automatically after certificate updates
- **VMware Integration**: Deploy certificates to ESXi hosts and restart management interfaces

## Prerequisites

### Required Software
- **certbot** - Let's Encrypt client for certificate acquisition
- **git** - Version control for certificate repository
- **openssl** - Certificate format conversion (PKCS12 generation)
- **keytool** (optional) - JKS keystore generation for Java applications
- **ssh/scp** - Remote deployment to VMware ESXi hosts (if using VMware features)

### DNS Requirements
- RFC2136-compliant DNS server with dynamic update capability (e.g., BIND9, PowerDNS)
- TSIG key configured for secure dynamic updates
- Proper DNS zone configuration allowing updates for certificate validation

### System Requirements
- Root or sudo access for certificate management
- `/opt/certificates` directory for certificate repository (or customize path)
- `/etc/letsencrypt/` directory for certbot configuration

## Installation

### 1. Clone Repository
```bash
git clone https://github.com/jrwoodman/rfc2136-cert-management.git
cd rfc2136-cert-management
```

### 2. Install Scripts
Copy scripts to system binary directory:
```bash
sudo cp scripts/* /usr/local/bin/
sudo chmod +x /usr/local/bin/{getcert,update_cert_repo,update_certs,update_vmware_certs}
```

### 3. Configure RFC2136 Credentials
Copy and edit the RFC2136 configuration file:
```bash
sudo mkdir -p /etc/letsencrypt
sudo cp config/rfc2136.ini /etc/letsencrypt/
sudo chmod 600 /etc/letsencrypt/rfc2136.ini
sudo vi /etc/letsencrypt/rfc2136.ini
```

Edit `rfc2136.ini` with your DNS server details:
```ini
dns_rfc2136_server = <your_dns_server_ip>
dns_rfc2136_port = 53
dns_rfc2136_name = <your_update_key_name>
dns_rfc2136_secret = <your_tsig_key_secret>
dns_rfc2136_algorithm = HMAC-SHA512
```

### 4. Initialize Certificate Repository
```bash
sudo mkdir -p /opt/certificates
cd /opt/certificates
sudo git init
```

### 5. Install Crontab (Optional)
For automated certificate management:
```bash
sudo cp crontab/root /etc/cron.d/certificates
# Or manually add entries to root's crontab:
sudo crontab -e
```

### 6. Configure VMware Integration (Optional)
If managing ESXi hosts:
```bash
# Edit update_vmware_certs with your ESXi hostnames
sudo vi /usr/local/bin/update_vmware_certs

# Copy esxi_update_ui_cert to each ESXi host
scp scripts/esxi_update_ui_cert root@your-esxi-host:~/
```

## Configuration

### Certificate Repository Path
Default: `/opt/certificates`

To change, edit the `CERTREPO` variable in:
- `getcert`
- `update_cert_repo`
- `update_certs`
- `update_vmware_certs`

### Service Integration
Edit `/usr/local/bin/update_certs` to uncomment and configure services that need certificate updates:

```bash
# Example: Enable Apache2 restart
/usr/bin/systemctl restart apache2
sleep ${DELAY}
```

Supported services include:
- Apache2
- audiobookshelf
- calibreweb
- Courier (IMAP/POP)
- Dovecot
- Lidarr
- nginx
- Postfix
- Radarr
- sabnzbdplus
- Sonarr (requires additional configuration for PKCS12 binding)

### VMware ESXi Hosts
Edit `/usr/local/bin/update_vmware_certs`:
```bash
VMSERVERS="esxi01.example.com esxi02.example.com"
VMUSER="root"
```

## Usage

### Manual Certificate Acquisition

Fetch a new certificate for a domain:
```bash
sudo getcert example.com
```

Or run interactively:
```bash
sudo getcert
# Enter domain when prompted
```

This will:
1. Request a wildcard certificate (`example.com` and `*.example.com`)
2. Store certificates in `/opt/certificates/example.com/`
3. Generate PKCS12 keystore (`.p12`)
4. Optionally generate JKS keystore (`.jks`)

### Automated Certificate Management

The cron jobs handle:

**Weekly Certificate Repository Update** (Fridays at 3:00 AM)
```bash
/usr/local/bin/update_cert_repo
```
- Pulls latest certificates from git repository
- Sets proper permissions on certificate files

**Weekly Local Certificate Update** (Fridays at 5:00 AM)
```bash
/usr/local/bin/update_certificates
```
- Updates local certificates
- Restarts configured services

**Weekly VMware Certificate Deployment** (Fridays at 5:00 AM)
```bash
/usr/local/bin/update_vmware_certs
```
- Deploys certificates to ESXi hosts
- Restarts ESXi management interface

## Certificate Formats

Each domain generates multiple certificate formats:

- **cert.pem** - Server certificate
- **chain.pem** - Certificate chain
- **fullchain.pem** - Full certificate chain (cert + chain)
- **privkey.pem** - Private key
- **domain.p12** - PKCS12 keystore (no password by default)
- **domain.jks** - Java KeyStore (optional, password-protected)

## Directory Structure

```
/opt/certificates/
├── example.com/
│   ├── cert.pem
│   ├── chain.pem
│   ├── fullchain.pem
│   ├── privkey.pem
│   ├── example.com.p12
│   └── example.com.jks
└── another-domain.com/
    └── ...

/etc/letsencrypt/
├── rfc2136.ini          # RFC2136 credentials
└── live/                # Certbot's certificate storage
```

## Security Considerations

### File Permissions
- RFC2136 credentials: `600` (root only)
- Certificate private keys: `400` or `440`
- Certificate repository: Appropriate permissions for accessing services

### TSIG Key Security
- Never commit `rfc2136.ini` to public repositories
- Use strong TSIG keys (SHA-256 or SHA-512)
- Restrict DNS update permissions to necessary zones only

### SSH Key Authentication
For VMware deployment, configure SSH key-based authentication:
```bash
ssh-copy-id root@esxi-host.example.com
```

## Caveats and Limitations

### Rate Limits
- Let's Encrypt has rate limits: 50 certificates per domain per week
- Be cautious with frequent certificate requests during testing
- Use Let's Encrypt staging environment for testing

### DNS Propagation
- RFC2136 updates must propagate before validation
- Default certbot timeout may be insufficient for some DNS servers
- Consider adding delays if DNS propagation is slow

### Wildcard Certificates
- Only DNS-01 challenge supports wildcard certificates
- Requires RFC2136 or other DNS API access
- Cannot use HTTP-01 challenge

### VMware ESXi Considerations
- ESXi certificate replacement requires management interface restart
- Brief service interruption during certificate update
- Ensure SSH access is configured and working
- `esxi_update_ui_cert` script must be in user's home directory on ESXi host

### Service Restart Timing
- Default 3-second delay between service restarts
- Adjust `DELAY` variable in `update_certs` if needed
- Some services may require longer restart times

### Git Repository
- Certificate repository should be private if shared
- Consider using git-crypt or similar for encrypted storage
- Certificates contain sensitive private key material

## Troubleshooting

### Certificate Acquisition Fails
1. Verify RFC2136 credentials are correct
2. Check DNS server allows updates for the zone
3. Test TSIG key with `nsupdate -k keyfile`
4. Review certbot logs: `/var/log/letsencrypt/`

### Services Not Restarting
1. Verify service names match your system
2. Check systemctl service status
3. Review system logs: `journalctl -xe`
4. Ensure services are configured to use certificate paths

### VMware Deployment Fails
1. Verify SSH connectivity to ESXi hosts
2. Check `esxi_update_ui_cert` is present in home directory
3. Ensure VMUSER has proper permissions
4. Verify certificate files are being copied correctly

### Permission Errors
1. Run scripts as root or with sudo
2. Check `/opt/certificates` ownership
3. Verify `/etc/letsencrypt/rfc2136.ini` is readable by root

## Contributing

Contributions are welcome! Please submit pull requests or open issues for:
- Additional service integrations
- Bug fixes
- Documentation improvements
- New features

## Support

For issues and questions:
- GitHub Issues: https://github.com/jrwoodman/rfc2136-cert-management/issues
- Review Let's Encrypt documentation: https://letsencrypt.org/docs/
- RFC2136 documentation: https://tools.ietf.org/html/rfc2136

## Acknowledgments

- Let's Encrypt for free SSL/TLS certificates
- Certbot team for excellent ACME client
- RFC2136 DNS update protocol
