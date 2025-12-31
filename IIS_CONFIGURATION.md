# IIS Host Configuration Guide

This guide explains how to configure IIS hosts to work with the Traefik load balancer setup.

## Architecture Overview

- **Traefik** terminates SSL from clients (uses Traefik's certificate)
- **Traefik** re-encrypts traffic to IIS backends (uses IIS certificates)
- **Sticky sessions** enabled via `traefik-sticky` cookie
- **Health checks** performed via HTTPS to `/` path

## IIS Configuration Requirements

### 1. SSL/TLS Certificate Configuration

Each IIS server needs its own SSL certificate for HTTPS communication with Traefik:

#### Install Certificate on IIS Server

1. **Import Certificate to Windows Certificate Store:**
   ```powershell
   # Import certificate (PFX format recommended)
   Import-PfxCertificate -FilePath "C:\path\to\certificate.pfx" -CertStoreLocation Cert:\LocalMachine\WebHosting -Password (ConvertTo-SecureString -String "YourPassword" -Force -AsPlainText)
   ```

2. **Or use Certificate Manager (MMC):**
   - Open `certlm.msc` (Local Computer)
   - Navigate to `Personal` → `Certificates`
   - Right-click → `All Tasks` → `Import`
   - Select your certificate file

#### Configure IIS Site Binding

1. Open **IIS Manager**
2. Select your website
3. Click **Bindings** in the right panel
4. Click **Add** or edit existing HTTPS binding:
   - **Type:** `https`
   - **IP address:** `All Unassigned` or specific IP (e.g., `192.168.1.10`)
   - **Port:** `443`
   - **SSL certificate:** Select your imported certificate
   - **Require Server Name Indication (SNI):** Unchecked (unless using SNI)
   - Click **OK**

### 2. Application Request Routing (ARR) Configuration

If you need to handle X-Forwarded headers or configure ARR:

#### Install ARR Module (if not already installed)

1. Download **Application Request Routing 3.0** from Microsoft
2. Install on each IIS server
3. Restart IIS: `iisreset`

#### Configure ARR Server Proxy Settings

1. In IIS Manager, select the **server node** (not a site)
2. Double-click **Application Request Routing Cache**
3. Click **Server Proxy Settings** in the right panel
4. Configure:
   - ✅ **Enable proxy:** Checked
   - ✅ **Reverse rewrite host in response headers:** Checked (important!)
   - **Response buffer threshold:** `0` (unlimited) or adjust as needed
   - Click **Apply**

#### Configure X-Forwarded Headers (Optional but Recommended)

If your application needs to know the original client IP:

1. Install **URL Rewrite Module 2.1** (if not installed)
2. In IIS Manager, select your **website**
3. Double-click **URL Rewrite**
4. Click **Add Rule** → **Blank Rule**
5. Configure:
   - **Name:** `X-Forwarded-For Header`
   - **Pattern:** `.*`
   - **Conditions:** Add condition:
     - **Condition input:** `{HTTP_X-Forwarded-For}`
     - **Check if input string:** `Does Not Match the Pattern`
     - **Pattern:** `.*`
   - **Server Variables:** Add:
     - **Name:** `REMOTE_ADDR`
     - **Value:** `{HTTP_X-Forwarded-For:0}`
   - Click **OK**

### 3. Sticky Session Configuration

Traefik uses a cookie named `traefik-sticky` for sticky sessions. IIS should:

#### Option A: Let Traefik Handle Sticky Sessions (Recommended)

- **No special IIS configuration needed**
- Traefik routes requests based on the `traefik-sticky` cookie
- IIS just needs to handle the cookie if your application uses it

#### Option B: Configure IIS Session Affinity (If Needed)

If you want additional session affinity at IIS level:

1. Install **Application Request Routing** (see above)
2. Configure **Server Farms** in ARR (if using multiple IIS servers behind another load balancer)
3. Enable **Server Affinity** with cookie-based affinity

**Note:** With Traefik handling sticky sessions, this is typically not needed.

### 4. Health Check Endpoint Configuration

Traefik performs health checks on the `/` path using HTTPS. Ensure:

1. **Root path (`/`) is accessible:**
   - Your IIS site should respond to requests to `/`
   - Default document should be configured (e.g., `default.aspx`, `index.html`)

2. **Health check returns 200 OK:**
   ```powershell
   # Test health check manually
   Invoke-WebRequest -Uri "https://192.168.1.10/" -UseBasicParsing
   ```

3. **Optional: Dedicated Health Check Endpoint**
   If you prefer a dedicated health check endpoint:
   
   - Create a simple page at `/health` or `/healthcheck`
   - Update `dynamic.yml` health check path:
     ```yaml
     healthCheck:
       path: "/health"
       interval: "30s"
       timeout: "3s"
       scheme: https
     ```

### 5. Security Configuration

#### TLS/SSL Settings

1. **Disable weak protocols:**
   ```powershell
   # Disable SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1 (keep only TLS 1.2+)
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Server" -Name "Enabled" -Value 0 -PropertyType DWORD -Force
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Server" -Name "Enabled" -Value 0 -PropertyType DWORD -Force
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" -Name "Enabled" -Value 0 -PropertyType DWORD -Force
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" -Name "Enabled" -Value 0 -PropertyType DWORD -Force
   ```

2. **Enable strong cipher suites:**
   - Ensure TLS 1.2 and TLS 1.3 are enabled
   - Use IISCrypto tool or Group Policy to configure cipher suites

#### Request Filtering

1. In IIS Manager, select your **website**
2. Double-click **Request Filtering**
3. Configure:
   - **Maximum allowed content length:** Adjust as needed
   - **Maximum URL length:** Adjust as needed
   - **Maximum query string:** Adjust as needed

#### IP Restrictions (Optional)

If you want to restrict access to Traefik only:

1. In IIS Manager, select your **website**
2. Double-click **IP Address and Domain Restrictions**
3. Click **Add Allow Entry**
4. Add Traefik server IP address(es)
5. Or add your internal network range (e.g., `192.168.1.0/24`)

### 6. Firewall Configuration

Ensure Windows Firewall allows HTTPS traffic:

```powershell
# Allow HTTPS (port 443) inbound
New-NetFirewallRule -DisplayName "HTTPS Inbound" -Direction Inbound -LocalPort 443 -Protocol TCP -Action Allow
```

Or configure via Windows Firewall GUI:
- **Inbound Rules** → **New Rule** → **Port** → **TCP** → **443** → **Allow**

### 7. Network Configuration

#### Ensure IIS Servers are Reachable

1. **Verify network connectivity from Traefik server:**
   ```bash
   # From Traefik server
   curl -k https://192.168.1.10:443/
   curl -k https://192.168.1.11:443/
   ```

2. **Verify DNS resolution** (if using hostnames instead of IPs)

3. **Check routing** between Traefik and IIS servers

### 8. Application-Specific Configuration

#### Handle X-Forwarded Headers in Application Code

If your application needs client IP or protocol information:

- **X-Forwarded-For:** Original client IP address
- **X-Forwarded-Proto:** Original protocol (usually `https`)
- **X-Forwarded-Host:** Original host header
- **X-Real-IP:** Alternative client IP header

Example (ASP.NET):
```csharp
string clientIp = Request.Headers["X-Forwarded-For"]?.FirstOrDefault() 
                  ?? Request.Headers["X-Real-IP"]?.FirstOrDefault()
                  ?? Request.UserHostAddress;
```

#### Session State Configuration

If using ASP.NET Session State:

1. **In-Memory Sessions:** Works with sticky sessions
2. **SQL Server Session State:** Works across all servers (no sticky session needed)
3. **State Server Session State:** Works across all servers

### 9. Monitoring and Logging

#### Enable IIS Logging

1. In IIS Manager, select your **website**
2. Double-click **Logging**
3. Configure:
   - **Format:** `W3C` (recommended)
   - **Directory:** Default or custom path
   - **Log file rollover:** Daily or by size

#### Monitor Health Check Responses

Check IIS logs for health check requests:
- Look for requests to `/` from Traefik server IP
- Should return `200 OK` status

### 10. Testing Configuration

#### Test from Traefik Server

```bash
# Test HTTPS connectivity
curl -k https://192.168.1.10:443/
curl -k https://192.168.1.11:443/

# Test with certificate validation
curl https://192.168.1.10:443/ --cacert /path/to/ca-cert.pem
```

#### Test Health Check

```powershell
# From IIS server or Traefik server
Invoke-WebRequest -Uri "https://192.168.1.10/" -UseBasicParsing
Invoke-WebRequest -Uri "https://192.168.1.11/" -UseBasicParsing
```

#### Test Full Request Flow

1. Make request to Traefik: `https://subdomain.host.com/`
2. Verify request reaches IIS backend
3. Check IIS logs for the request
4. Verify response is returned correctly

## Troubleshooting

### Common Issues

1. **Certificate Errors:**
   - Verify certificate is installed correctly
   - Check certificate is bound to port 443
   - Ensure certificate is not expired
   - Verify certificate matches the server's hostname/IP

2. **Connection Refused:**
   - Check Windows Firewall allows port 443
   - Verify IIS is running
   - Check binding is configured correctly
   - Verify network connectivity

3. **Health Check Failing:**
   - Ensure `/` path is accessible
   - Check IIS site is started
   - Verify SSL certificate is valid
   - Check IIS logs for errors

4. **Sticky Sessions Not Working:**
   - Verify Traefik sticky cookie is being set
   - Check browser is accepting cookies
   - Verify application handles cookies correctly

5. **X-Forwarded Headers Missing:**
   - Install URL Rewrite Module
   - Configure ARR server proxy settings
   - Check application code reads headers correctly

## Summary Checklist

- [ ] SSL certificate installed on each IIS server
- [ ] HTTPS binding configured on port 443
- [ ] TLS 1.2+ enabled, weak protocols disabled
- [ ] Health check endpoint (`/`) returns 200 OK
- [ ] Windows Firewall allows port 443
- [ ] Network connectivity verified from Traefik to IIS
- [ ] IIS logging enabled
- [ ] Application configured to handle X-Forwarded headers (if needed)
- [ ] Session state configured appropriately
- [ ] All IIS servers have consistent configuration

## Additional Resources

- [IIS SSL Configuration](https://docs.microsoft.com/en-us/iis/manage/configuring-security/how-to-set-up-ssl-on-iis)
- [Application Request Routing](https://docs.microsoft.com/en-us/iis/extensions/planning-for-arr/using-the-application-request-routing-module)
- [URL Rewrite Module](https://docs.microsoft.com/en-us/iis/extensions/url-rewrite-module/using-the-url-rewrite-module)

