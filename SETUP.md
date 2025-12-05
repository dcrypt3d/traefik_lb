# Traefik NIST-Aligned Security Setup Guide

## Quick Start

### 1. Configure Server Details

Edit `dynamic.yml` directly to configure your servers. Update the following sections:

**Router rule** (line ~12):
```yaml
rule: "HostRegexp(`{subdomain:[a-z0-9-]+}.host.com`)"
```
Replace `host.com` with your domain.

**Backend servers** (lines ~38-40):
```yaml
servers:
  - url: "https://192.168.1.10:443"
  - url: "https://192.168.1.11:443"
  - url: "https://192.168.1.12:443"
```
Replace with your actual server IPs, ports, and protocols.

**Health check** (lines ~43-45):
```yaml
healthCheck:
  path: "/"
  interval: "30s"
  timeout: "3s"
```
Adjust the health check path and timing as needed.

### 2. Create Required Directories

```bash
# Create certificates directory
mkdir certs

# Create logs directory for audit logs
mkdir logs

# Set proper permissions
chmod 700 certs
chmod 755 logs
```

### 3. Add SSL Certificates

Place your SSL certificates in the `certs` directory:
- `certs/cert.pem` - Your SSL certificate
- `certs/key.pem` - Your private key

Set secure permissions:
```bash
chmod 600 certs/key.pem
chmod 644 certs/cert.pem
```

**Note:** Your certificate should cover `*.yourdomain.com` (wildcard certificate) or include all the specific subdomains you plan to use.

### 4. Configure IP Whitelisting (Optional)

If you need to restrict access, edit `dynamic.yml` and uncomment the `ip-whitelist` middleware, then configure your allowed IP ranges.

### 5. Start Traefik

```bash
docker-compose up -d
```

Traefik will use the configuration values directly from `dynamic.yml`.

### 6. Verify Logging

Check that logs are being written:
```bash
tail -f logs/traefik.log
tail -f logs/access.log
```

## Configuration Files

- `traefik.yml` - Static configuration with TLS, logging, and security settings
- `dynamic.yml` - Dynamic configuration with routes, services, and middlewares
- `docker-compose.yml` - Container orchestration with security constraints

## How It Works

1. Docker Compose starts the Traefik container with mounted configuration files
2. Traefik reads `dynamic.yml` with your server and routing configuration
3. Configuration updates when you restart the container after editing files

## Security Features Enabled

✅ TLS 1.2+ with strong cipher suites
✅ Security headers (HSTS, XSS protection, etc.)
✅ Rate limiting (100 req/s)
✅ Structured JSON logging for SIEM
✅ Access logging with security fields
✅ Resource limits
✅ No new privileges
✅ Health checks

## Next Steps

1. Review `NIST_SECURITY.md` for detailed security documentation
2. Configure IP whitelisting if needed
3. Set up log rotation for the logs directory
4. Integrate logs with your SIEM system
5. Configure monitoring and alerting

## Troubleshooting

### Logs not being written
- Ensure `logs` directory exists and is writable
- Check Docker volume permissions

### Certificate errors
- Verify certificates are in `certs/` directory
- Check file permissions (key should be 600)
- Ensure certificate matches your domain configured in `dynamic.yml`

### Rate limiting too strict
- Adjust limits in `dynamic.yml` under `rate-limit` middleware

### Configuration changes not applied
- Restart Traefik after editing configuration: `docker-compose restart`
