# Traefik Configuration for Dynamic Domain Routing

This configuration routes HTTPS requests for `*.host.com` domains to internal IIS hosts using **SSL termination with re-encryption**. Traefik terminates SSL from clients, processes the traffic, then re-encrypts it to backend servers.

## SSL Termination with Re-encryption

This configuration uses **SSL termination** at Traefik, which means:
- Traefik terminates SSL from clients using your certificate
- Traefik can inspect and modify HTTP traffic
- Traefik re-encrypts traffic to backend servers using HTTPS
- All HTTP-level features are available

**Features Enabled:**
- ✅ **Sticky sessions** (cookie-based routing) - Same client always goes to same backend
- ✅ **HTTP-level middlewares** (security headers, rate limiting, IP whitelisting, etc.)
- ✅ **HTTP authentication** and authorization
- ✅ **Path-based routing**
- ✅ **Request/response modification**
- ✅ **Load balancing** with health checks
- ✅ **Security headers** automatically added to responses

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

**Backend servers** - Update IPs and ports (HTTPS for re-encryption):
```yaml
servers:
  - url: "https://192.168.1.10:443"
  - url: "https://192.168.1.11:443"
```

**Sticky sessions** - Enabled by default (cookie-based):
```yaml
sticky:
  cookie:
    name: traefik-sticky
    secure: true
    httpOnly: true
    sameSite: none
```

**Health check** - HTTPS health checks work properly:
```yaml
healthCheck:
  path: "/"
  interval: "30s"
  timeout: "3s"
  scheme: https
```

**Note:** Sticky sessions are enabled by default. Each client will be routed to the same backend server based on the `traefik-sticky` cookie.

### 2. Configure Custom Certificate

Place your certificate and private key files in the certs directory:

- Certificate file: `./certs/cert.pem`
- Private key file: `./certs/key.pem`

**Note:** Your certificate should cover `*.yourdomain.com` (wildcard certificate) or include all the specific subdomains you plan to use. This certificate is used for SSL termination (client → Traefik).

**Backend Certificates:** Your backend servers should have their own valid HTTPS certificates. Traefik will validate these certificates when re-encrypting traffic to backends.

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

- **SSL Termination**: Traefik terminates SSL from clients using your certificate
- **Re-encryption**: Traffic is re-encrypted to backend servers using HTTPS
- **Sticky Sessions**: Cookie-based session affinity ensures same client → same backend
- Automatic HTTP to HTTPS redirect
- **Load Balancing**: Round-robin distribution across multiple HTTPS backend servers
- **Security Headers**: Automatically adds security headers to all responses
- **Rate Limiting**: HTTP-level rate limiting (100 req/s average, 50 burst)
- **Health Checks**: HTTPS health checks every 30 seconds
- Custom SSL certificate support for client connections
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

- **SSL Termination**: Traefik decrypts client traffic, enabling all HTTP-level features
- **Re-encryption**: Traffic to backends is re-encrypted using HTTPS (backend certificates)
- The configuration uses a regex rule to match any subdomain of your domain
- All HTTP traffic is automatically redirected to HTTPS
- **Sticky Sessions**: Enabled by default - clients are routed to the same backend server via cookies
- Health checks are performed every 30 seconds using HTTPS (configurable in `dynamic.yml`)
- Make sure your certificate file covers `*.yourdomain.com` (wildcard) or includes all subdomains you plan to use
- Backend servers must have valid HTTPS certificates (Traefik validates them)
- To change configuration, edit `dynamic.yml` and restart Traefik: `docker-compose restart`
- Dashboard and API are disabled for security
- Security headers middleware is automatically applied to all responses

## Security Features

- Read-only root filesystem
- No new privileges
- Resource limits (CPU and memory)
- NIST-aligned TLS configuration (TLS 1.2+, strong cipher suites)
- **Security headers** automatically added (XSS protection, frame options, content type sniffing prevention, etc.)
- **Rate limiting** to prevent abuse (100 requests/second average)
- Structured logging for SIEM integration
- Syslog support for centralized log management
- Dashboard and API disabled
- HTTPS re-encryption to backend servers ensures end-to-end encryption

