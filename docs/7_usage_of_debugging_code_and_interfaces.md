# Usage of Debugging Code and Interfaces

## Introduction

Debug interfaces, diagnostic code, and development features are essential during product development but represent **critical security vulnerabilities** if left active in production firmware. In 2025, with the EU Cyber Resilience Act and increasing regulatory scrutiny, removal of debug code from release builds is not just a best practice—it's a compliance requirement.

This chapter addresses three major threat vectors:

1. **Leftover debug code** in firmware (console access, diagnostic commands, verbose logging)
2. **Hardware debug interfaces** (JTAG, UART, SWD left enabled in production)
3. **ODM/Third-party backdoors** (intentional or accidental, increasingly common in supply chain)
4. **HTTP debug endpoints** (hidden web interfaces providing elevated access)

These vulnerabilities have been exploited in high-profile attacks, including the 2016 Mirai botnet (default credentials on debug interfaces) and numerous CVEs related to undisclosed admin panels in consumer routers and IoT devices.

---

## ODM/OEM Supply Chain Security Risks

Original Design Manufacturers (ODMs) and contract manufacturers frequently introduce **hidden backdoors** for their own support and factory testing purposes. These backdoors often persist into mass-production firmware, creating systemic vulnerabilities across entire product lines.

### The ODM Threat Model

**Scenario**: An OEM contracts an ODM in Asia to design and manufacture a smart camera. The ODM develops firmware with various debug features for factory testing and remote support.

**Common ODM Backdoors**:

1. **Hidden Diagnostic Endpoints**:
   * `/debug.cgi` - Command execution via HTTP
   * `/telnet_enable.cgi` - Enables telnet on port 23
   * `/factory_reset.php` - Bypass authentication
   * `/engineer_mode` - Undocumented admin panel

2. **Hardcoded ODM Credentials**:
   * Username: `engineer` / Password: `ODM_Model_2024`
   * Username: `factory` / Password: `12345678`
   * SSH keys for ODM support staff in `authorized_keys`

3. **Factory Test Interfaces**:
   * UART console with root shell (no password)
   * JTAG debugging left enabled
   * Special boot mode via GPIO pins

4. **Remote Support Backdoors**:
   * Cloud callback to ODM servers (`support.odmvendor.com`)
   * VPN tunnels for remote access
   * Hardcoded certificates for ODM infrastructure

### Real-World Examples (Anonymized)

**Case Study 1: Router with Hidden Telnet Activation** (2018-2023)

A popular consumer router brand (ODM-manufactured) contained a hidden CGI script:

```bash
# GET /telnet_enable.cgi HTTP/1.1
# Response: Telnet enabled on port 23

# Discovered via:
$ binwalk -e firmware.bin
$ grep -r "telnet" _firmware.extracted/squashfs-root/www/cgi-bin/
```

**Impact**: Any local network attacker could enable telnet and gain root access.

**Case Study 2: IP Camera with Hardcoded ODM Credentials** (2020-2024)

Firmware analysis revealed:

```c
// /bin/webserver binary (decompiled with Ghidra)
if (strcmp(username, "engineer") == 0 &&
    strcmp(password, "CameraODM2020!") == 0) {
    grant_root_access();
    // ODM support account, forgot to remove
}
```

**Discovery Method**:
```bash
strings /bin/webserver | grep -i password
# Output: "CameraODM2020!"
```

**Case Study 3: Smart Thermostat with Debug API** (2022-2025)

Hidden HTTP API discovered:

```http
POST /api/v1/debug/exec HTTP/1.1
Content-Type: application/json

{
  "cmd": "id; cat /etc/shadow"
}

HTTP/1.1 200 OK
{
  "output": "uid=0(root) gid=0(root)\nroot:$6$..."
}
```

**Discovery**: Directory fuzzing with SecLists wordlist:
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u https://thermostat.local/FUZZ
# Found: /api/v1/debug/
```

### Supply Chain Threat Actors

| Actor Type | Motivation | Typical Backdoors | Risk Level |
|------------|-----------|-------------------|------------|
| **ODM Engineering** | Factory testing convenience | UART root shells, telnet activation | High |
| **ODM Support Team** | Remote customer support | SSH keys, VPN tunnels | Critical |
| **Third-Party Contractors** | Legacy debug code | Diagnostic web pages, verbose logging | Medium |
| **Malicious Insiders** | Espionage, sabotage | Covert backdoors, exfiltration | Critical |
| **Compromised Build Systems** | Supply chain attack (e.g., SolarWinds-style) | Injected malware, persistent access | Critical |

### Contractual Protections (MSA Requirements)

**Master Service Agreement (MSA) Security Clauses**:

```markdown
## ODM/Contractor Security Requirements (Template)

### 1. No Backdoor Code
Contractor warrants that all delivered software is free from:
- Undocumented administrative accounts or credentials
- Hidden network services or debug interfaces
- Unauthorized remote access mechanisms
- Hardcoded passwords or cryptographic keys

### 2. Secure Development Lifecycle
Contractor shall:
- Follow OWASP Embedded Application Security Best Practices
- Conduct SAST/DAST security testing before delivery
- Provide source code for OEM security audit
- Remove all debug code from production builds

### 3. Code Review Rights
OEM retains rights to:
- Review all source code and binaries
- Perform penetration testing on deliverables
- Require remediation of identified vulnerabilities
- Terminate contract for security violations

### 4. Audit Trail
Contractor must provide:
- Software Bill of Materials (SBOM) in SPDX/CycloneDX format
- Build environment documentation
- List of all developer/support accounts
- Cryptographic key management records

### 5. Liability and Indemnification
Contractor liable for:
- Security vulnerabilities in delivered code
- Regulatory fines resulting from backdoors
- Customer data breaches traced to contractor code
- Remediation costs (up to 3x contract value)

### 6. Secure Handoff Process
Before mass production, contractor must:
- Provide signed attestation of no backdoors
- Participate in OEM security audit
- Transfer all credentials and keys to OEM
- Delete all support access mechanisms

### 7. Post-Delivery Support
Contractor shall:
- Provide security updates for [X] years
- Respond to CVE disclosures within 72 hours
- Maintain security point-of-contact
- Report any discovered vulnerabilities immediately
```

### Pre-Deployment Validation Process

**Step 1: Firmware Extraction and Analysis**

```bash
#!/bin/bash
# ODM firmware security audit script

FIRMWARE="odm_delivered_firmware.bin"

echo "[*] Extracting filesystem..."
binwalk -e $FIRMWARE

FS_ROOT="_${FIRMWARE}.extracted/squashfs-root"

echo "[*] Searching for suspicious strings..."
strings -n 8 $FS_ROOT/bin/* $FS_ROOT/sbin/* > strings_output.txt

# Check for common backdoor patterns
grep -E "(telnet|debug|factory|engineer|backdoor|hidden)" strings_output.txt

echo "[*] Searching for hardcoded credentials..."
grep -rE "(password|passwd|pwd|secret|key).*=" $FS_ROOT/etc/
grep -rE "admin|root|support" $FS_ROOT/etc/passwd $FS_ROOT/etc/shadow

echo "[*] Searching for HTTP debug endpoints..."
find $FS_ROOT/www -name "*.cgi" -o -name "*.php" -o -name "*.asp" | xargs grep -l "debug\|diag\|factory\|test"

echo "[*] Checking for UART/serial access..."
grep -r "console=ttyS" $FS_ROOT/boot/

echo "[*] Searching for SSH authorized_keys..."
find $FS_ROOT -name "authorized_keys" -exec cat {} \;

echo "[*] Checking for outbound connections..."
grep -rE "connect|curl|wget" $FS_ROOT/etc/init.d/ $FS_ROOT/etc/rc.d/
```

**Step 2: Binary Analysis (Automated)**

```python
#!/usr/bin/env python3
# firmware_backdoor_scanner.py

import re
import subprocess
import sys

SUSPICIOUS_PATTERNS = [
    b"telnet_enable",
    b"hidden_admin",
    b"debug_mode",
    b"factory_reset",
    b"engineer_mode",
    b"ODM.*password",
    b"backdoor",
    b"root.*123456"
]

def scan_binary(binary_path):
    # Run strings
    result = subprocess.run(['strings', binary_path], capture_output=True)
    strings_output = result.stdout

    findings = []
    for pattern in SUSPICIOUS_PATTERNS:
        matches = re.findall(pattern, strings_output, re.IGNORECASE)
        if matches:
            findings.append(f"Found suspicious pattern in {binary_path}: {pattern}")

    return findings

if __name__ == "__main__":
    binary = sys.argv[1]
    results = scan_binary(binary)
    for finding in results:
        print(f"[!] {finding}")
```

**Step 3: Dynamic Analysis (Runtime Testing)**

```bash
# Boot firmware in QEMU and test for backdoors
qemu-system-arm -M virt -kernel zImage -initrd rootfs.cpio.gz \
                -append "console=ttyAMA0" -nographic

# Test telnet access
nmap -p 23,2323,9000-9999 192.168.1.100

# Fuzz web interface for hidden endpoints
wfuzz -c -z file,/usr/share/wordlists/dirb/common.txt \
      --hc 404 http://192.168.1.100/FUZZ

# Test default credentials
hydra -L usernames.txt -P passwords.txt http-get://192.168.1.100/admin
```

### Detection and Mitigation

**Build-Time Controls**:

```makefile
# Yocto recipe to strip debug code
do_install:append() {
    # Remove debug binaries
    rm -f ${D}${bindir}/telnetd ${D}${sbindir}/dropbear

    # Remove debug scripts
    find ${D}/www -name "*debug*" -delete
    find ${D}/www -name "*factory*" -delete

    # Strip binaries
    ${STRIP} ${D}${bindir}/*

    # Remove UART console from kernel cmdline
    sed -i 's/console=ttyS0//' ${D}/boot/uEnv.txt
}
```

**Runtime Detection** (in production firmware):

```c
// Detect unauthorized network services
#include <stdio.h>
#include <stdlib.h>

void check_unauthorized_services(void) {
    FILE *fp;
    char line[256];

    // Check for telnet
    fp = popen("netstat -tuln | grep :23", "r");
    if (fgets(line, sizeof(line), fp) != NULL) {
        syslog(LOG_CRIT, "SECURITY: Unauthorized telnet service detected!");
        system("killall telnetd");  // Emergency mitigation
    }
    pclose(fp);

    // Check for unauthorized HTTP endpoints
    fp = popen("curl -I http://localhost/debug.cgi 2>&1", "r");
    if (fgets(line, sizeof(line), fp) != NULL && strstr(line, "200 OK")) {
        syslog(LOG_CRIT, "SECURITY: Debug CGI endpoint active!");
    }
    pclose(fp);
}
```

---

## HTTP Debug Endpoints - Discovery and Exploitation

Hidden HTTP debug interfaces are among the most common vulnerabilities in embedded web servers, often introduced by ODMs or left over from development.

### Common Vulnerable Endpoint Patterns

#### 1. Directory Traversal to Hidden Paths

**Vulnerable URL Structure**:
```
/admin/              → 403 Forbidden (redirects to login)
/admin_backup/       → 200 OK (no auth required!)
/admin/../debug/     → 200 OK (path traversal bypass)
```

**Real-World Discovery Example**:
```bash
# Directory fuzzing
gobuster dir -u http://router.local -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Results:
#   /admin         (Status: 302) [Redirect to /login]
#   /factory       (Status: 200) [Hidden factory panel]
#   /engineering   (Status: 200) [Debug interface]
#   /backup        (Status: 200) [Config backup, no auth]
```

#### 2. Command Injection via Debug Parameters

**Vulnerable Code** (common in CGI scripts):

```php
<?php
// /www/cgi-bin/diagnostic.php (ODM debug page)
if (isset($_GET['cmd'])) {
    // ❌ DANGEROUS: No authentication, no input validation
    $output = shell_exec($_GET['cmd']);
    echo "<pre>$output</pre>";
}
?>
```

**Exploitation**:
```bash
curl "http://device.local/cgi-bin/diagnostic.php?cmd=id"
# Output: uid=0(root) gid=0(root)

curl "http://device.local/cgi-bin/diagnostic.php?cmd=cat%20/etc/shadow"
# Full system compromise
```

#### 3. Hidden API Endpoints

**Vulnerable REST API**:

```python
# Flask debug API (left in production build)
from flask import Flask, request
import subprocess

app = Flask(__name__)

@app.route('/api/debug/exec', methods=['POST'])
def debug_exec():
    # ❌ No authentication, command injection
    cmd = request.json.get('cmd')
    result = subprocess.check_output(cmd, shell=True)
    return {'output': result.decode()}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=True)  # ❌ Debug mode on!
```

**Discovery**:
```bash
# API endpoint fuzzing
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
     -u http://device.local/api/FUZZ -mc 200

# Test found endpoint
curl -X POST http://device.local/api/debug/exec \
     -H "Content-Type: application/json" \
     -d '{"cmd": "whoami"}'
```

#### 4. Debug Mode Activation via Headers

**Vulnerable Code**:

```c
// Embedded lighttpd server module
if (getenv("HTTP_X_DEBUG_MODE")) {
    enable_debug_features();  // ❌ Enables verbose logging, SQL dumps
}

if (strcmp(getenv("HTTP_USER_AGENT"), "DevTools/1.0") == 0) {
    bypass_authentication();  // ❌ Secret ODM bypass
}
```

**Exploitation**:
```bash
curl -H "X-Debug-Mode: 1" http://device.local/admin
# Verbose error messages leak internal paths

curl -A "DevTools/1.0" http://device.local/admin
# Authentication bypassed
```

### Discovery Techniques (for Security Testing)

#### Firmware-Based Discovery

```bash
#!/bin/bash
# Extract firmware and search for debug endpoints

binwalk -e firmware.bin
FS_ROOT="_firmware.bin.extracted/squashfs-root"

echo "[*] Searching for web files with 'debug' in name..."
find $FS_ROOT/www -iname "*debug*" -o -iname "*diag*" -o -iname "*factory*"

echo "[*] Searching for command execution in web code..."
grep -rE "(system|exec|popen|shell_exec|eval)\s*\(" $FS_ROOT/www/ | grep -v ".js"

echo "[*] Searching for authentication bypasses..."
grep -rE "(bypass|skip.*auth|debug.*mode)" $FS_ROOT/www/

echo "[*] Extracting URL paths from JavaScript..."
grep -roE "/(api|admin|debug|factory|diag|eng|test)/[a-zA-Z0-9_/]+" $FS_ROOT/www/js/ | sort -u
```

#### Network-Based Discovery

**Comprehensive Web Fuzzing**:

```bash
# Fuzz for hidden directories
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
     -u http://target/FUZZ -fc 404,403

# Fuzz for hidden files
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt \
     -u http://target/FUZZ -fc 404 -e .php,.cgi,.asp,.jsp

# Fuzz API endpoints
ffuf -w /path/to/api-wordlist.txt \
     -u http://target/api/v1/FUZZ -mc 200,500

# Check for debug headers
curl -H "X-Debug: 1" -v http://target/
curl -H "X-Debug-Mode: true" -v http://target/
curl -H "X-Developer: 1" -v http://target/
```

**Automated Scanner Integration**:

```bash
# OWASP ZAP automated scan
zap-cli quick-scan -s all -r report.html http://target/

# Nikto web server scanner
nikto -h http://target/ -C all -o nikto_results.txt

# Custom Nmap NSE scripts
nmap --script http-enum,http-methods,http-put --script-args http-enum.basepath=/admin target
```

### Secure Implementation Examples

**✅ SECURE: CGI Diagnostic Endpoint with Proper Controls**

```c
// /www/cgi-bin/diagnostics.c - Secure implementation

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <syslog.h>

// Whitelist of allowed diagnostic commands
static const char *ALLOWED_COMMANDS[] = {
    "/usr/bin/uptime",
    "/usr/bin/free -m",
    "/usr/bin/df -h",
    NULL  // Terminator
};

int is_command_allowed(const char *cmd) {
    for (int i = 0; ALLOWED_COMMANDS[i] != NULL; i++) {
        if (strcmp(cmd, ALLOWED_COMMANDS[i]) == 0) {
            return 1;
        }
    }
    return 0;
}

int main(void) {
    char *query_string = getenv("QUERY_STRING");
    char *session_token = getenv("HTTP_AUTHORIZATION");

    // 1. Require authentication
    if (!session_token || !verify_admin_session(session_token)) {
        printf("Status: 401 Unauthorized\r\n\r\n");
        syslog(LOG_WARNING, "Unauthorized diagnostic access attempt from %s",
               getenv("REMOTE_ADDR"));
        return 1;
    }

    // 2. Parse and validate command parameter
    char cmd[256] = {0};
    if (sscanf(query_string, "cmd=%255s", cmd) != 1) {
        printf("Status: 400 Bad Request\r\n\r\n");
        return 1;
    }

    // 3. Whitelist validation (no arbitrary commands)
    if (!is_command_allowed(cmd)) {
        printf("Status: 403 Forbidden\r\n\r\n");
        syslog(LOG_ALERT, "Blocked unauthorized diagnostic command: %s from %s",
               cmd, getenv("REMOTE_ADDR"));
        return 1;
    }

    // 4. Execute safe command
    printf("Content-Type: text/plain\r\n\r\n");
    FILE *fp = popen(cmd, "r");
    if (fp) {
        char line[256];
        while (fgets(line, sizeof(line), fp)) {
            printf("%s", line);
        }
        pclose(fp);
    }

    // 5. Audit log
    syslog(LOG_INFO, "Admin %s executed diagnostic: %s",
           get_username_from_session(session_token), cmd);

    return 0;
}
```

**✅ SECURE: Python Flask API with Proper Security**

```python
from flask import Flask, request, jsonify
from functools import wraps
import subprocess
import logging
import hmac

app = Flask(__name__)

# Security configuration
API_KEY_HASH = "sha256_hash_of_secret_key"  # From environment variable
ALLOWED_COMMANDS = [
    "/usr/bin/uptime",
    "/usr/bin/free -m",
    "/usr/bin/df -h"
]

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        if not api_key or not hmac.compare_digest(
                hashlib.sha256(api_key.encode()).hexdigest(),
                API_KEY_HASH):
            logging.warning(f"Unauthorized API access from {request.remote_addr}")
            return jsonify({'error': 'Unauthorized'}), 401
        return f(*args, **kwargs)
    return decorated_function

@app.route('/api/diagnostics', methods=['POST'])
@require_api_key
def diagnostics():
    cmd = request.json.get('command')

    # Whitelist validation
    if cmd not in ALLOWED_COMMANDS:
        logging.alert(f"Blocked unauthorized command: {cmd}")
        return jsonify({'error': 'Command not allowed'}), 403

    # Execute with timeout
    try:
        result = subprocess.run(cmd.split(),
                               capture_output=True,
                               timeout=5,
                               text=True)

        logging.info(f"Diagnostic command executed: {cmd}")
        return jsonify({'output': result.stdout})

    except subprocess.TimeoutExpired:
        return jsonify({'error': 'Command timeout'}), 500

if __name__ == '__main__':
    # ✅ Production configuration
    app.run(host='127.0.0.1',  # Localhost only, not 0.0.0.0
            port=8080,
            debug=False,        # Debug mode OFF
            use_reloader=False)
```

### Mitigation Strategies

#### Build-Time Removal

```makefile
# Makefile rule to remove debug endpoints
RELEASE_BUILD ?= 1

ifeq ($(RELEASE_BUILD),1)
install-web:
	# Install production web files
	cp -r www/public/* $(DESTDIR)/www/

	# ✅ Remove debug endpoints
	rm -rf $(DESTDIR)/www/debug
	rm -rf $(DESTDIR)/www/factory
	rm -rf $(DESTDIR)/www/diagnostics
	rm -f $(DESTDIR)/www/cgi-bin/*debug*
	rm -f $(DESTDIR)/www/cgi-bin/*factory*

	# ✅ Strip debug symbols from CGI binaries
	find $(DESTDIR)/www/cgi-bin -type f -exec strip {} \;

	echo "Debug endpoints removed from release build"
else
	# Development build includes debug endpoints
	cp -r www/* $(DESTDIR)/www/
endif
```

**Yocto Recipe Example**:

```bitbake
# meta-mylayer/recipes-web/webserver/webserver_1.0.bb

do_install:append() {
    # Remove debug content for production images
    if [ "${IMAGE_FEATURES}" != *"debug-tweaks"* ]; then
        rm -rf ${D}/www/debug
        rm -rf ${D}/www/factory
        rm -f ${D}/www/cgi-bin/*_debug.*

        bbwarn "Debug web endpoints removed for production build"
    fi
}
```

#### Runtime Protection

**Web Application Firewall (WAF) Rules**:

```nginx
# nginx configuration with ModSecurity WAF

location ~* \/(debug|factory|diag|test|admin_backup)\/ {
    # Block access to known debug paths
    deny all;
    return 403;
}

location ~ \.(php|cgi|asp)$ {
    # Require authentication for dynamic content
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # Rate limiting
    limit_req zone=dynamic burst=5;

    fastcgi_pass unix:/var/run/php-fpm.sock;
}

# ModSecurity rules
modsecurity on;
modsecurity_rules '
    SecRule REQUEST_URI "@contains debug" "id:1001,deny,log,msg:\'Debug endpoint blocked\'"
    SecRule REQUEST_URI "@contains factory" "id:1002,deny,log,msg:\'Factory endpoint blocked\'"
    SecRule ARGS:cmd "@pmFromFile /etc/modsecurity/command-injection.txt" "id:1003,deny,log"
';
```

**Application-Level Checks**:

```c
// Runtime detection of unauthorized endpoints
#include <dirent.h>
#include <syslog.h>

void audit_web_endpoints(void) {
    DIR *dir;
    struct dirent *ent;

    const char *FORBIDDEN_PATTERNS[] = {
        "debug", "diag", "factory", "test", "engineer", NULL
    };

    dir = opendir("/www");
    if (dir) {
        while ((ent = readdir(dir)) != NULL) {
            for (int i = 0; FORBIDDEN_PATTERNS[i] != NULL; i++) {
                if (strstr(ent->d_name, FORBIDDEN_PATTERNS[i]) != NULL) {
                    syslog(LOG_CRIT,
                           "SECURITY ALERT: Forbidden debug file found: /www/%s",
                           ent->d_name);

                    // Optional: Auto-delete (risky, log only in production)
                    // unlink(ent->d_name);
                }
            }
        }
        closedir(dir);
    }
}
```

---

## Hardware Debug Interfaces

In addition to software debug code, physical debug interfaces (JTAG, UART, SWD) must be secured or disabled in production devices.

### Common Hardware Debug Interfaces

| Interface | Purpose | Production Risk | Mitigation |
|-----------|---------|-----------------|------------|
| **JTAG** | Boundary scan, flash programming | Firmware extraction, modification | Disable via eFuses, require authentication |
| **UART** | Serial console, bootloader access | Root shell access | Disable console in kernel cmdline, require password |
| **SWD** (ARM) | Debugging Cortex-M | Flash dumping, code injection | Debug Access Port (DAP) lock, TrustZone |
| **I2C/SPI** | Bus communication | Secure element communication sniffing | Encrypted communication, physical tamper detection |

### Secure UART Configuration

**❌ Insecure** (common in development):
```bash
# /boot/uEnv.txt
console=ttyS0,115200  # Full root shell on UART!
```

**✅ Secure** (production):
```bash
# Disable UART console entirely
# (remove console=ttyS0 from kernel cmdline)

# OR require authentication:
# /etc/inittab
ttyS0::respawn:/sbin/getty -L 115200 ttyS0 vt100 -l /bin/login_uart
```

**Secure UART Login Script**:
```bash
#!/bin/sh
# /bin/login_uart - Authenticated UART access

MAX_ATTEMPTS=3
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
    echo -n "Password: "
    read -s PASSWORD
    echo

    # Hash and compare (never plaintext passwords)
    HASH=$(echo -n "$PASSWORD" | sha256sum | cut -d' ' -f1)
    if [ "$HASH" = "expected_sha256_hash" ]; then
        logger "UART authentication successful"
        exec /bin/sh
    fi

    ATTEMPT=$((ATTEMPT + 1))
    echo "Authentication failed"
    sleep 3  # Rate limiting
done

logger "UART authentication failed after $MAX_ATTEMPTS attempts from $(tty)"
exit 1
```

### JTAG Security

**Disable JTAG via eFuses** (NXP i.MX example):
```bash
# Burn eFuse to permanently disable JTAG (IRREVERSIBLE)
# See processor reference manual for specific fuse addresses

# For development: Conditional JTAG (requires signed challenge)
# Implement JTAG authentication in secure boot ROM
```

**ARM Debug Authentication**:
```c
// ARM Cortex-A Debug Access Port (DAP) authentication

// Configure Debug Authentication Control Register (DBGDSAR)
// Require authentication key for debug access

void secure_debug_config(void) {
    // Enable authentication required for invasive debug
    write_cp15_register(DBGDSAR, 0xC5ACCE55);  // Magic unlock value

    // Set authentication key (from secure storage)
    write_cp15_register(DBGDSMR, get_debug_auth_key());

    // Lock configuration (prevents modification)
    write_cp15_register(DBGDLAR, 0xC5ACCE55);
}
```

---

## Integration with OWASP IoT Security Ecosystem

### OWASP FSTM Stage 7: Runtime Analysis

This chapter's content directly supports **OWASP Firmware Security Testing Methodology (FSTM) Stage 7**:

* **FSTM-7.1**: Identify debug interfaces (UART, JTAG, SWD)
* **FSTM-7.2**: Test for hidden web endpoints
* **FSTM-7.3**: Analyze for ODM backdoors
* **FSTM-7.4**: Test command injection in diagnostic features

### OWASP ISTG Test Cases

* **ISTG-FW-INFO-003**: Information Gathering - Debug Code
* **ISTG-WRLS-JTAG**: Hardware Debug Interface Testing
* **ISTG-UI**: Web User Interface Security Testing

---

## Best Practices Checklist

### Pre-Release Security Audit

- [ ] All ODM-delivered firmware binaries analyzed for backdoors
- [ ] Firmware extracted and `strings` analysis performed
- [ ] Web directories fuzzed for hidden debug endpoints
- [ ] UART console tested (should require authentication or be disabled)
- [ ] JTAG/SWD access tested (should be disabled or require auth)
- [ ] All debug logging disabled (no verbose error messages)
- [ ] Network services audited (`netstat -tuln` shows only required ports)
- [ ] Default credentials removed (test with common password lists)
- [ ] Third-party binaries scanned with antivirus/malware scanners
- [ ] SBOM generated and all components CVE-checked

### Build System Controls

- [ ] Separate debug and release build configurations
- [ ] Automated stripping of debug symbols from binaries
- [ ] Debug endpoints removed at build time (not runtime checks)
- [ ] Code signing enforced (prevents post-build modification)
- [ ] Build reproducibility (hash verification of builds)

### Runtime Monitoring

- [ ] Audit log for unauthorized access attempts
- [ ] Network traffic monitoring for unexpected connections
- [ ] File integrity monitoring (detect backdoor installation)
- [ ] Intrusion detection system (IDS) for embedded devices

---

It is important to ensure all unnecessary pre-production build code, as well as dead and unused code, has been removed prior to firmware release to all market segments. This includes but is not limited to potential backdoor code and root privilege accounts that may have been left by parties such as Original Design Manufacturers (ODM) and Third-Party contractors. Typically this falls in scope for Original Equipment Manufacturers (OEM) to perform via contracts or reverse engineering of binaries. This should also require ODMs to sign Master Service Agreements (MSA) insuring that either no backdoor code is included and that all code has been reviewed for software security vulnerabilities holding all Third-Party developers accountable for devices that are mass deployed into the market.

**General Considerations (Disclaimer: The List below is non-exhaustive):**

* Remove backdoor accounts used for debugging, deployment verification and/or customer support purposes.
* Ensure development, diagnostic, or debug features are not included within release builds.
* Perform code cleanup sessions to ensure dead or unused code is removed across repositories.
* Ensure third-party libraries and binary images are reviewed for backdoors by staff before market deployment.
* Implement automated scanning in CI/CD pipelines to detect debug code.
* Require security audits of all ODM/contractor deliverables.
* Maintain Software Bill of Materials (SBOM) for all components.

---

## Tools for Debug Code Detection

### Static Analysis Tools

* **Binwalk** - Firmware extraction and filesystem analysis
* **Ghidra** - NSA's reverse engineering framework (free, open-source)
* **radare2/rizin** - Open-source reverse engineering framework
* **IDA Pro** - Commercial disassembler (industry standard)
* **strings** - Extract printable strings from binaries
* **grep** - Search for suspicious patterns in code
* **Semgrep** - Static analysis for security patterns
* **Flawfinder / rats** - C/C++ security scanners

### Dynamic Analysis Tools

* **QEMU** - Emulate embedded firmware for testing
* **Firmadyne** - Automated firmware emulation and testing
* **Firmware Analysis Toolkit (FAT)** - Wrapper for Firmadyne
* **FACT** - Firmware Analysis and Comparison Tool
* **qiling** - Advanced binary emulation framework

### Web Security Tools

* **ffuf** - Fast web fuzzer
* **gobuster** - Directory/DNS busting tool
* **Burp Suite** - Web vulnerability scanner (commercial/community)
* **OWASP ZAP** - Free web application security scanner
* **Nikto** - Web server scanner
* **wfuzz** - Web application fuzzer

### Network Scanning

* **nmap** - Network discovery and security auditing
* **Wireshark** - Network protocol analyzer
* **tcpdump** - Command-line packet analyzer

---

## Additional References

### Official Standards and Guidelines

* **ETSI EN 303 645 Provision 6**: Minimize exposed attack surfaces
* **ETSI EN 303 645 Provision 2**: Implement vulnerability disclosure policy
* **EU Cyber Resilience Act Annex I**: No known exploitable vulnerabilities
* **OWASP FSTM Stage 7**: Runtime analysis and debug interface testing
* **OWASP ISTG**: IoT Security Testing Guide

### Vulnerability Databases

* [OWASP Leftover Debug Code](https://owasp.org/www-community/vulnerabilities/Leftover_Debug_Code)
* [CWE-489: Active Debug Code](https://cwe.mitre.org/data/definitions/489.html)
* [CWE-489: Leftover Debug Code](https://cwe.mitre.org/data/definitions/489.html)
* [VU#419568: CERT Coordination Center - Backdoors in embedded devices](http://www.kb.cert.org/vuls/id/419568)

### Tools and Resources

* [OWASP FSTM Firmware Analysis Tool Index](https://scriptingxss.gitbook.io/firmware-security-testing-methodology/#firmware-and-binary-analysis-tool-index)
* [SecLists Wordlists](https://github.com/danielmiessler/SecLists) - For fuzzing
* [Ghidra](https://ghidra-sre.org/) - NSA reverse engineering tool
* [radare2](https://rada.re/) - Open-source reverse engineering framework
* [Binwalk](https://github.com/ReFirmLabs/binwalk) - Firmware analysis tool
* [FACT](https://fkie-cad.github.io/FACT_core/) - Firmware Analysis and Comparison Tool

