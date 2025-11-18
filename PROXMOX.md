# Proxmox ACME Certificate Setup with RFC2136

This guide walks through configuring Proxmox VE to automatically obtain and renew SSL/TLS certificates from Let's Encrypt using RFC2136 DNS validation.

## Overview

Proxmox VE includes built-in ACME (Automatic Certificate Management Environment) support. Using RFC2136 DNS validation allows you to:
- Obtain wildcard certificates
- Validate domains without exposing HTTP services
- Use certificates on servers behind firewalls
- Automate certificate renewal

## Prerequisites

### DNS Server Requirements
- RFC2136-compliant DNS server (BIND9, PowerDNS, etc.)
- TSIG key configured for dynamic DNS updates
- Proper zone configuration allowing updates

### Proxmox Requirements
- Proxmox VE 6.2 or later
- Root or administrative access to Proxmox web interface
- Fully qualified domain name (FQDN) configured for Proxmox host

### Information Needed
- DNS server IP address
- DNS server port (usually 53)
- TSIG key name
- TSIG key secret
- TSIG algorithm (e.g., HMAC-SHA512, HMAC-SHA256)

## DNS Server Configuration

### BIND9 Example

1. **Generate TSIG Key**
```bash
tsig-keygen -a HMAC-SHA512 acme-update > /etc/bind/keys/acme-update.key
```

2. **Configure Zone to Allow Updates**

Edit your zone file configuration (e.g., `/etc/bind/named.conf.local`):
```
key "acme-update" {
    algorithm hmac-sha512;
    secret "YOUR_BASE64_ENCODED_SECRET";
};

zone "example.com" {
    type master;
    file "/var/lib/bind/example.com.zone";
    allow-update { key acme-update; };
};
```

3. **Restart BIND**
```bash
systemctl restart bind9
```

4. **Test Dynamic Updates**
```bash
nsupdate -k /etc/bind/keys/acme-update.key
> server your-dns-server
> zone example.com
> update add _acme-challenge.test.example.com. 300 TXT "test-value"
> send
> quit

# Verify the update
dig @your-dns-server _acme-challenge.test.example.com TXT

# Clean up test record
nsupdate -k /etc/bind/keys/acme-update.key
> server your-dns-server
> zone example.com
> update delete _acme-challenge.test.example.com. TXT
> send
> quit
```

### PowerDNS Example

1. **Enable DNS Update in PowerDNS**

Edit `/etc/powerdns/pdns.conf`:
```
dnsupdate=yes
allow-dnsupdate-from=127.0.0.1,YOUR_PROXMOX_IP
```

2. **Create TSIG Key**
```bash
pdnsutil generate-tsig-key acme-update hmac-sha512
```

3. **Associate Key with Zone**
```bash
pdnsutil activate-tsig-key example.com acme-update master
pdnsutil set-meta example.com ALLOW-DNSUPDATE-FROM AUTO-NS
pdnsutil set-meta example.com TSIG-ALLOW-DNSUPDATE acme-update
```

## Proxmox Configuration

### Step 1: Add ACME Account

1. Log into Proxmox web interface as root
2. Navigate to **Datacenter** → **ACME**
3. Click **Add** to create a new ACME account
4. Configure:
   - **Account Name**: `default` (or your preferred name)
   - **E-Mail**: Your email address for Let's Encrypt notifications
   - **ACME Directory**: `Let's Encrypt V2` (production) or `Let's Encrypt V2 Staging` (testing)
5. Accept Terms of Service
6. Click **Register**

### Step 2: Add DNS Challenge Plugin

1. Still in **Datacenter** → **ACME**
2. Click **Challenge Plugins** tab
3. Click **Add**
4. Configure:
   - **Plugin ID**: `rfc2136` (or your preferred name)
   - **DNS API**: Select `RFC2136`
   - **API Data**:
     ```
     NSUPDATE_SERVER=your-dns-server-ip
     NSUPDATE_KEY=acme-update
     NSUPDATE_ZONE=example.com
     ```
   - **Validation Delay**: `30` (seconds, adjust based on DNS propagation time)

5. Click **Add**

### Step 3: Configure TSIG Key File

SSH into your Proxmox host and create the TSIG key file:

```bash
# Create directory for ACME DNS credentials
mkdir -p /etc/pve/priv/acme

# Create TSIG key file
cat > /etc/pve/priv/acme/rfc2136.key <<EOF
key "acme-update" {
    algorithm hmac-sha512;
    secret "YOUR_BASE64_ENCODED_SECRET";
};
EOF

# Set proper permissions
chmod 600 /etc/pve/priv/acme/rfc2136.key
```

**Note**: The key name must match what you configured in your DNS server.

### Step 4: Configure Node Certificate

1. Navigate to your Proxmox node (e.g., **Datacenter** → **pve01**)
2. Go to **System** → **Certificates**
3. Click **Add** under ACME
4. Configure:
   - **Challenge Type**: `DNS`
   - **Plugin**: Select your RFC2136 plugin
   - **Domain**: Your Proxmox host FQDN (e.g., `pve01.example.com`)
5. Click **Add**

### Step 5: Optional - Add Wildcard Domain

To add a wildcard certificate:

1. Click **Add** again under ACME
2. Configure:
   - **Challenge Type**: `DNS`
   - **Plugin**: Select your RFC2136 plugin
   - **Domain**: `*.example.com`
3. Click **Add**

### Step 6: Order Certificate

1. In **System** → **Certificates**
2. Click **Order Certificates Now**
3. Monitor the task log for progress
4. Certificate will be automatically obtained and installed

## Alternative Configuration Method (CLI)

You can also configure ACME via command line:

```bash
# Add ACME account
pvenode acme account register default your-email@example.com

# Add DNS plugin
pvenode acme plugin add dns rfc2136 \
  --data "NSUPDATE_SERVER=your-dns-server-ip" \
  --data "NSUPDATE_KEY=acme-update" \
  --data "NSUPDATE_ZONE=example.com" \
  --validation-delay 30

# Add domain to node configuration
pvenode config set --acme domains=pve01.example.com,plugin=rfc2136

# Order certificate
pvenode acme cert order
```

## Automatic Renewal

Proxmox automatically renews certificates through a systemd timer:

```bash
# Check renewal timer status
systemctl status pve-daily-update.timer

# View last renewal attempt
journalctl -u pve-daily-update.service

# Manually trigger renewal
pvenode acme cert renew
```

Certificates are checked daily and renewed when they have less than 30 days remaining.

## Verification

### Check Certificate Status

**Via Web Interface:**
1. Navigate to node → **System** → **Certificates**
2. View certificate details and expiration date

**Via Command Line:**
```bash
# View certificate information
pvenode cert info

# Check certificate expiration
openssl x509 -in /etc/pve/nodes/$(hostname)/pveproxy-ssl.pem -noout -dates
```

### Test HTTPS Access

```bash
# Test certificate from another machine
openssl s_client -connect pve01.example.com:8006 -servername pve01.example.com

# Check certificate in browser
# Navigate to: https://pve01.example.com:8006
```

## Troubleshooting

### Certificate Order Fails

**Check ACME logs:**
```bash
journalctl -u pve-daily-update.service -n 100
cat /var/log/syslog | grep -i acme
```

**Common Issues:**

1. **DNS not updating**
   - Verify TSIG key is correct
   - Check DNS server allows updates from Proxmox IP
   - Test with `nsupdate` manually

2. **Validation timeout**
   - Increase validation delay in plugin settings
   - Check DNS propagation time: `dig @8.8.8.8 _acme-challenge.pve01.example.com TXT`
   - Verify DNS server is publicly accessible

3. **TSIG key file not found**
   - Ensure `/etc/pve/priv/acme/rfc2136.key` exists
   - Check file permissions (should be 600)
   - Verify key name matches DNS configuration

### DNS Update Not Working

```bash
# Test DNS update manually
nsupdate -k /etc/pve/priv/acme/rfc2136.key <<EOF
server your-dns-server
zone example.com
update add _acme-challenge.pve01.example.com. 300 TXT "test"
send
EOF

# Check if record was created
dig @your-dns-server _acme-challenge.pve01.example.com TXT
```

### Rate Limit Errors

Let's Encrypt has rate limits:
- 50 certificates per registered domain per week
- 5 duplicate certificates per week

**Solutions:**
- Use Let's Encrypt Staging for testing
- Wait for rate limit to reset
- Check rate limit status: https://crt.sh

### Certificate Not Applied

```bash
# Restart Proxmox web services
systemctl restart pveproxy
systemctl restart pvedaemon

# Force certificate reload
pvenode cert info
```

## Security Best Practices

### TSIG Key Security
- Use strong algorithms (HMAC-SHA512 or HMAC-SHA256)
- Keep TSIG keys private and secure
- Don't commit keys to version control
- Use restrictive file permissions (600)

### DNS Server Security
- Restrict DNS updates to specific zones
- Limit update access to Proxmox IP addresses
- Use firewall rules to protect DNS server
- Monitor DNS logs for unauthorized updates

### Network Security
- Use private network for DNS updates if possible
- Consider VPN for remote Proxmox nodes
- Implement firewall rules between Proxmox and DNS server

## Cluster Considerations

For Proxmox clusters:

1. **Configure each node separately** - Each node needs its own certificate
2. **Use node-specific FQDNs** - `pve01.example.com`, `pve02.example.com`, etc.
3. **Alternative: Wildcard certificate** - Use `*.example.com` for all nodes
4. **Shared configuration** - ACME plugin configuration is cluster-wide
5. **Individual renewal** - Each node renews its certificate independently

### Example Multi-Node Setup

```bash
# Node 1
pvenode config set --acme domains=pve01.example.com,plugin=rfc2136

# Node 2
pvenode config set --acme domains=pve02.example.com,plugin=rfc2136

# Node 3
pvenode config set --acme domains=pve03.example.com,plugin=rfc2136

# Order certificates on each node
pvenode acme cert order
```

## Additional Resources

- **Proxmox ACME Documentation**: https://pve.proxmox.com/wiki/Certificate_Management
- **Let's Encrypt Documentation**: https://letsencrypt.org/docs/
- **RFC2136 Specification**: https://tools.ietf.org/html/rfc2136
- **BIND9 TSIG Documentation**: https://bind9.readthedocs.io/en/latest/
- **Let's Encrypt Rate Limits**: https://letsencrypt.org/docs/rate-limits/

## See Also

- Main repository README for general certificate management
- BIND9 or PowerDNS documentation for DNS server configuration
- Proxmox certificate management for custom certificate installations
