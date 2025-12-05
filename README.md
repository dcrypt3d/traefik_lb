# Traefik Configuration for Dynamic Domain Routing

This configuration routes HTTPS requests for `*.host.com` domains to internal IIS hosts using **TLS passthrough**. This means Traefik forwards encrypted traffic directly to backend servers without decrypting it.

## Important: TLS Passthrough Limitations

With TLS passthrough, Traefik cannot inspect or modify HTTP traffic, so the following features are **not available**:
- HTTP-level middlewares (security headers, HTTP rate limiting, etc.)
- Sticky sessions (cookie-based routing)
- HTTP authentication
- Path-based routing
- Request/response modification

**What DOES work:**
- TCP-level load balancing
- Health checks (if backend supports)
- TCP-level rate limiting (via TCP router)
- TCP-level IP whitelisting (via TCP router)

## Files

- `traefik.yml` - Static Traefik configuration (TLS, logging, security settings)
- `dynamic.yml` - Dynamic routing configuration (edit directly)
- `docker-compose.yml` - Docker Compose configuration with security constraints

## Configuration

### 1. Configure Server Details

Edit `dynamic.yml` directly to configure your domain and backend servers:

**Router rule** - Update domain:
```yaml
rule: "HostRegexp(`{subdomain:[a-z0-9-]+}.host.com`)"
```

**Backend servers** - Update IPs and ports:
```yaml
servers:
  - url: "https://192.168.1.10:443"
  - url: "https://192.168.1.11:443"
```

**Health check** - Adjust as needed (may be limited with TLS passthrough):
```yaml
healthCheck:
  path: "/"
  interval: "30s"
  timeout: "3s"
  scheme: https
```

**Note:** Sticky sessions are disabled because they don't work with TLS passthrough. Traefik cannot set/modify cookies in encrypted traffic.

### 2. Configure Custom Certificate

Place your certificate and private key files in the certs directory:

- Certificate file: `./certs/cert.pem`
- Private key file: `./certs/key.pem`

**Note:** Your certificate should cover `*.yourdomain.com` (wildcard certificate) or include all the specific subdomains you plan to use.

## Usage

### Docker Compose

Start Traefik using Docker Compose:

```bash
docker-compose up -d
```

**Key Configuration Details:**
- Uses `network_mode: host` for direct network access
- Read-only root filesystem enabled for security
- Dashboard and API are disabled
- Syslog logging configured (update `syslog-address` in `docker-compose.yml` to point to your syslog server)
- Resource limits: 2 CPU cores, 2GB RAM max
- Security constraints: no-new-privileges enabled

**Note:** This configuration is compatible with Traefik v3.x. For the latest version, check the [official Traefik releases](https://github.com/traefik/traefik/releases).

## Features

- **TLS Passthrough**: Encrypted traffic forwarded directly to backend servers
- Automatic HTTP to HTTPS redirect
- Custom SSL certificate support (for SNI routing)
- Load balancing across IIS hosts (TCP-level, no sticky sessions)
- Health checks for backend servers (may be limited with TLS passthrough)
- NIST-aligned security configuration
- Structured JSON logging (file-based and syslog)
- Read-only filesystem for enhanced security
- Resource limits to prevent resource exhaustion
- Supports all subdomains of your configured domain (e.g., app.host.com, api.host.com, etc.)

## Logging

Logs are written to:
- **File-based**: `./logs/traefik.log` and `./logs/access.log` (JSON format)
- **Syslog**: Configured in `docker-compose.yml` (default: `tcp://localhost:514`)

Update the `syslog-address` in `docker-compose.yml` to point to your syslog server.

## Notes

- **TLS Passthrough**: Traefik does not decrypt traffic, so HTTP-level features are unavailable
- The configuration uses a regex rule to match any subdomain of your domain
- All HTTP traffic is automatically redirected to HTTPS
- Health checks are performed every 30 seconds (configurable in `dynamic.yml`)
- **No sticky sessions**: Cookie-based routing doesn't work with TLS passthrough
- Make sure your certificate file covers `*.yourdomain.com` (wildcard) or includes all subdomains you plan to use
- To change configuration, edit `dynamic.yml` and restart Traefik: `docker-compose restart`
- Dashboard and API are disabled for security

## Security Features

- Read-only root filesystem
- No new privileges
- Resource limits (CPU and memory)
- NIST-aligned TLS configuration (TLS 1.2+, strong cipher suites)
- Structured logging for SIEM integration
- Syslog support for centralized log management
- Dashboard and API disabled

