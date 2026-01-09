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

ARR is essential when IIS servers are behind a reverse proxy (like Traefik) to properly handle forwarded headers and ensure correct request/response processing.

#### 2.1 Install ARR Module

**Prerequisites:**
- Windows Server with IIS 7.0 or later
- Administrator privileges
- Internet connection (for download)

**Installation Steps:**

1. **Download Application Request Routing 3.0:**
   - Direct download: [ARR 3.0](https://www.iis.net/downloads/microsoft/application-request-routing)
   - Or search for "Application Request Routing 3.0" on Microsoft IIS Downloads
   - File name: `requestRouter_amd64.msi` (for 64-bit) or `requestRouter_x86.msi` (for 32-bit)

2. **Install via GUI:**
   - Run the downloaded `.msi` file
   - Click **Next** through the installation wizard
   - Accept the license agreement
   - Click **Install**
   - Wait for installation to complete
   - Click **Finish**

3. **Install via PowerShell (Automated):**
   ```powershell
   # Download ARR 3.0
   $arrUrl = "https://download.microsoft.com/download/4/9/C/49CD28DB-4AA6-4A51-9437-AA001221F606/requestRouter_amd64.msi"
   $arrInstaller = "$env:TEMP\requestRouter_amd64.msi"
   Invoke-WebRequest -Uri $arrUrl -OutFile $arrInstaller
   
   # Install silently
   Start-Process msiexec.exe -ArgumentList "/i $arrInstaller /quiet /norestart" -Wait
   
   # Restart IIS
   iisreset
   ```

4. **Verify Installation:**
   - Open **IIS Manager**
   - Select the **server node** (top-level, not a site)
   - Look for **Application Request Routing Cache** icon in the Features View
   - If present, installation was successful

5. **Restart IIS:**
   ```powershell
   iisreset
   ```

#### 2.2 Configure ARR Server Proxy Settings

These settings configure ARR to act as a reverse proxy backend, properly handling requests from Traefik.

**Step-by-Step Configuration:**

1. **Open IIS Manager:**
   - Press `Win + R`, type `inetmgr`, press Enter
   - Or search for "Internet Information Services (IIS) Manager"

2. **Select Server Node:**
   - In the left Connections pane, click on the **server name** (e.g., `SERVER01 (SERVER01\Administrator)`)
   - **Important:** Do NOT select a website - select the server itself

3. **Open ARR Configuration:**
   - In the Features View (center pane), double-click **Application Request Routing Cache**
   - You should see options in the right Actions pane

4. **Configure Server Proxy Settings:**
   - In the right Actions pane, click **Server Proxy Settings...**
   - The **Application Request Routing** dialog opens

5. **Configure Proxy Settings:**
   
   **Enable Proxy:**
   - ✅ Check **Enable proxy**
   - This allows IIS to act as a reverse proxy backend

   **Reverse Rewrite Host in Response Headers:**
   - ✅ Check **Reverse rewrite host in response headers**
   - **Critical:** This ensures IIS rewrites the `Host` header in responses to match what Traefik expects
   - Without this, Traefik may receive incorrect host headers

   **Response Buffer Threshold:**
   - Set to `0` (unlimited) for most scenarios
   - Or set a specific value (in bytes) if you need to limit buffering
   - Example: `10485760` (10 MB) for large file handling

   **Connection Timeout:**
   - Default: `90` seconds
   - Adjust if you have long-running requests
   - Recommended: `90` to `300` seconds

   **Preserve Client IP:**
   - ✅ Check **Preserve client IP** (if available)
   - This helps maintain original client IP information

6. **Apply Settings:**
   - Click **Apply** in the Actions pane
   - Settings are saved immediately

**Configuration via PowerShell (Alternative Method):**

```powershell
# Import WebAdministration module
Import-Module WebAdministration

# Enable proxy
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "enabled" -Value $true

# Enable reverse rewrite host
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "reverseRewriteHostInResponseHeaders" -Value $true

# Set response buffer threshold (0 = unlimited)
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "responseBufferThreshold" -Value 0

# Set connection timeout (in seconds)
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "timeout" -Value "00:01:30"

# Restart IIS to apply changes
iisreset
```

**Configuration via web.config (Machine-Level):**

Edit `C:\Windows\System32\inetsrv\config\applicationHost.config`:

```xml
<system.webServer>
  <proxy enabled="true" 
         reverseRewriteHostInResponseHeaders="true" 
         responseBufferThreshold="0" />
</system.webServer>
```

#### 2.3 Configure X-Forwarded Headers Handling

When Traefik forwards requests to IIS, it adds X-Forwarded-* headers. Your IIS application needs to read these to get the original client information.

**Install URL Rewrite Module 2.1 (Required):**

1. **Download URL Rewrite Module:**
   - Download from: [URL Rewrite Module 2.1](https://www.iis.net/downloads/microsoft/url-rewrite)
   - File: `rewrite_amd64_en-US.msi`

2. **Install:**
   ```powershell
   # Download
   $rewriteUrl = "https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi"
   $rewriteInstaller = "$env:TEMP\rewrite_amd64_en-US.msi"
   Invoke-WebRequest -Uri $rewriteUrl -OutFile $rewriteInstaller
   
   # Install
   Start-Process msiexec.exe -ArgumentList "/i $rewriteInstaller /quiet /norestart" -Wait
   iisreset
   ```

**Method 1: Configure X-Forwarded-For via URL Rewrite (Recommended)**

This method sets the `REMOTE_ADDR` server variable from the `X-Forwarded-For` header so your application can read the original client IP.

1. **Open IIS Manager**
2. **Select your website** (not the server node)
3. **Open URL Rewrite:**
   - Double-click **URL Rewrite** in Features View
4. **Add Inbound Rule:**
   - Click **Add Rule(s)...** in the right Actions pane
   - Select **Blank Rule**
   - Click **OK**
5. **Configure Rule:**
   - **Name:** `Set Remote Addr from X-Forwarded-For`
   - **Requested URL:** Matches the Pattern
   - **Using:** Regular Expressions
   - **Pattern:** `.*` (matches all URLs)
   - **Ignore case:** Checked
6. **Add Condition:**
   - Expand **Conditions** section
   - Click **Add...**
   - **Condition input:** `{HTTP_X-Forwarded-For}`
   - **Check if input string:** Matches the Pattern
   - **Pattern:** `.*` (matches any value)
   - Click **OK**
7. **Set Server Variable:**
   - Expand **Server Variables** section
   - Click **Add...**
   - **Server variable name:** `REMOTE_ADDR`
   - **Value:** `{HTTP_X-Forwarded-For:0}` (takes first IP from comma-separated list)
   - Click **OK**
8. **Configure Action:**
   - Expand **Action** section
   - **Action type:** Select **None**
   - **Why "None"?** Since we're only setting a server variable (REMOTE_ADDR) and not rewriting the URL or redirecting the request, we use "None". This allows the rule to set the variable and then continue processing the request normally to your application.
   - **Alternative actions (not needed here):**
     - **Rewrite:** Used when you want to change the URL path
     - **Redirect:** Used when you want to send a redirect response to the client
     - **None:** Used when you only want to set server variables or conditions without URL modification
9. **Apply:**
   - Click **Apply** in the right Actions pane

**Method 2: Configure via web.config (Site-Level)**

Add to your website's `web.config`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="Set Remote Addr from X-Forwarded-For" stopProcessing="false">
          <match url=".*" />
          <conditions>
            <add input="{HTTP_X-Forwarded-For}" pattern=".*" />
          </conditions>
          <serverVariables>
            <set name="REMOTE_ADDR" value="{HTTP_X-Forwarded-For:0}" />
          </serverVariables>
          <action type="None" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

**Note:** To use server variables in rewrite rules, you must unlock them first:

```powershell
# Unlock REMOTE_ADDR server variable
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/rewrite/allowedServerVariables" -Name "." -Value @{name="REMOTE_ADDR"}

# Unlock other X-Forwarded headers if needed
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/rewrite/allowedServerVariables" -Name "." -Value @{name="HTTP_X-Forwarded-For"}
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/rewrite/allowedServerVariables" -Name "." -Value @{name="HTTP_X-Forwarded-Proto"}
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/rewrite/allowedServerVariables" -Name "." -Value @{name="HTTP_X-Forwarded-Host"}
```

#### 2.4 Configure ARR Cache Settings (Optional)

ARR includes caching capabilities. For backend servers behind Traefik, caching is typically disabled, but you can configure it if needed.

1. **Open ARR Cache Settings:**
   - In IIS Manager, select **server node**
   - Double-click **Application Request Routing Cache**
   - Click **Cache** in the right Actions pane

2. **Configure Cache:**
   - **Enable disk cache:** Unchecked (for backend servers)
   - **Cache size limit:** Leave default or set to `0` to disable
   - **Maximum response size:** Adjust based on your needs

**Disable Cache via PowerShell:**
```powershell
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/cache" -Name "enabled" -Value $false
```

#### 2.5 Verify ARR Configuration

**Check Proxy Settings:**
```powershell
# Verify proxy is enabled
Get-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "enabled"

# Verify reverse rewrite host is enabled
Get-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy" -Name "reverseRewriteHostInResponseHeaders"
```

**Test X-Forwarded Headers:**
```powershell
# Test with X-Forwarded-For header
$headers = @{
    "X-Forwarded-For" = "192.168.1.100"
    "X-Forwarded-Proto" = "https"
    "X-Forwarded-Host" = "subdomain.host.com"
}
Invoke-WebRequest -Uri "https://localhost/" -Headers $headers -UseBasicParsing
```

**Check IIS Logs:**
- Open IIS logs: `C:\inetpub\logs\LogFiles\W3SVC<site-id>\`
- Look for requests with X-Forwarded-For headers
- Verify REMOTE_ADDR is being set correctly

#### 2.6 ARR Troubleshooting

**Issue: Proxy not working**
- Verify ARR is installed: Check for "Application Request Routing Cache" in IIS Manager
- Verify proxy is enabled: Check Server Proxy Settings
- Restart IIS: `iisreset`
- Check Windows Event Viewer for errors

**Issue: Host header incorrect in responses**
- Ensure "Reverse rewrite host in response headers" is checked
- Verify Traefik is sending correct Host header
- Check IIS logs for incoming Host headers

**Issue: X-Forwarded-For not working**
- Verify URL Rewrite Module is installed
- Check server variables are unlocked (see PowerShell commands above)
- Verify rewrite rule is configured correctly
- Test with manual request including X-Forwarded-For header

**Issue: Application can't read client IP**
- Ensure REMOTE_ADDR server variable is set from X-Forwarded-For
- Check application code reads from correct source:
  - ASP.NET: `Request.ServerVariables["REMOTE_ADDR"]` or `Request.UserHostAddress`
  - Or read directly: `Request.Headers["X-Forwarded-For"]`

**Enable ARR Logging:**
```powershell
# Enable failed request tracing for ARR
Set-WebConfigurationProperty -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/tracing/traceFailedRequests" -Name "enabled" -Value $true
```

**Check ARR Status:**
```powershell
# View ARR configuration
Get-WebConfiguration -PSPath "MACHINE/WEBROOT/APPHOST" -Filter "system.webServer/proxy"
```

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

