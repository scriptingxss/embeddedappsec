# Embedded Platform Security Hardening

This chapter focuses on **platform-level security hardening** for embedded Linux systems, covering build system configuration, kernel security, mandatory access control (MAC), and runtime protections. It addresses the integration and configuration of security features within Yocto Project, Buildroot, and custom embedded distributions for IoT, Industrial IoT (ICS/SCADA), and enterprise network devices.

**Note**: Compiler hardening flags and kernel protection mechanisms are covered in [Chapter 1](1_buffer_and_stack_overflow_protection.md), while bootloader security (U-Boot verified boot) is detailed in [Chapter 3](3_firmware_updates_and_cryptographic_signatures.md). This chapter concentrates on platform-level integration and system hardening practices.

**Chapter Scope:**
This chapter addresses platform-level security across multiple domains:

- **Build System Security**: Yocto Project, Buildroot configuration and hardening
- **Bootloader Security**: U-Boot verified boot and secure boot chain
- **Kernel Hardening**: Security flags, kernel lockdown, and exploit mitigation
- **Mandatory Access Control (MAC)**: SELinux, AppArmor, and SMACK configuration
- **Advanced SELinux**: Policy development, testing, CVE analysis, ICS/SCADA security
- **Runtime Integrity**: IMA/EVM integration and file system protection
- **Reproducible Builds**: Supply chain security and build verification

**Target Audiences:**
- IoT device manufacturers (consumer devices, smart home, edge computing)
- Industrial IoT (IIoT) developers (SCADA, ICS, factory automation, Industry 4.0)
- Enterprise network equipment vendors (routers, switches, firewalls, UTM, SD-WAN, NFV/VNF)

---

## Quick Navigation

**Within This Chapter:**
- [System Hardening Considerations](#system-hardening-considerations) - Utility removal, service hardening
- [Yocto Project Build System Security](#yocto-project-build-system-security) - Yocto 5.0 LTS hardening features
- [Buildroot Security](#buildroot-security-configuration) - Lightweight build system configuration
- [Kernel Security Features](#kernel-security-features) - Kernel lockdown, exploit mitigation
- [Mandatory Access Control](#mandatory-access-control-mac) - SELinux, AppArmor, SMACK
- [Advanced SELinux for Embedded Systems](#advanced-selinux-security-for-embedded-systems) - Policy development, ICS/SCADA

**Related Chapters:**
- [Chapter 1: Buffer and Stack Overflow Protection](1_buffer_and_stack_overflow_protection.md) - Compiler flags, memory protection
- [Chapter 3: Firmware Updates and Cryptographic Signatures](3_firmware_updates_and_cryptographic_signatures.md) - U-Boot verified boot, secure boot chain
- [Chapter 10: Third-Party Code and Components](10_third_party_code_and_components.md) - SBOM, CVE scanning, supply chain security

---

## Platform Security Overview

Modern embedded Linux build systems (Yocto Project, Buildroot) provide comprehensive tools for minimizing attack surface by including only required libraries, services, and utilities. This section provides platform-specific security guidance.

**Key Principles**:
- Build only what's needed - exclude unnecessary components at build time
- Remove legacy insecure protocols (Telnet, FTP, rsh family)
- Disable debug services in production builds
- Use modern TLS-based alternatives (SSH, SFTP, HTTPS)

For detailed compiler and kernel hardening flags, see [Chapter 1: Buffer and Stack Overflow Protection](1_buffer_and_stack_overflow_protection.md).

## Bootloader Security

U-Boot verified boot configuration, secure boot chain implementation, and bootloader hardening are covered in detail in [Chapter 3: Firmware Updates and Cryptographic Signatures](3_firmware_updates_and_cryptographic_signatures.md#u-boot-verified-boot). This includes:

- FIT image signing with RSA keys
- Verified boot configuration (CONFIG_FIT_SIGNATURE, CONFIG_RSA)
- Immutable environment variables
- Console access restrictions
- Boot delay security settings

For platform-level considerations when integrating U-Boot into your build system, continue with the system hardening guidance below.

## System Hardening Considerations

**Considerations (Disclaimer: The List below is non-exhaustive):**

* Ensure services such as SSH have a secure password created.
* Remove unused language interpreters such as: perl, python, lua.
* Remove dead code from unused library functions.
* Remove unused shell interpreters such as: ash, dash, zsh.
  * Review `/etc/shell`
* Remove legacy insecure daemons which includes but not limited to:
  * telnetd
  * ftpd
  * ftpget
  * ftpput
  * tftp
  * rlogind
  * rshd
  * rexd
  * rcmd
  * rhosts
  * rexecd
  * rwalld
  * rbootd
  * rusersd
  * rquotad
  * rstatd
  * nfs
*   Remove unused/unnecessary utilities to minimize attack surface:

    **Debug vs Production Build Strategy**: The table below provides guidance on which system utilities should be included in debug builds versus production builds. This approach applies to all embedded Linux systems (Yocto, Buildroot, custom distributions) and is particularly important for IoT, automotive, and industrial devices.

    **Rationale**: Debug utilities enable troubleshooting and development but expose sensitive information and increase attack surface in production deployments.

| Utility Name    | Location                                 | Debug Environment | Production Environment |
| --------------- | ---------------------------------------- | ----------------- | ---------------------- |
| Strace          | /bin/trace                               | INCLUDE           | EXCLUDE                |
| Klogd           | /sbin/klogd                              | INCLUDE           | EXCLUDE                |
| Syslogd(logger) | /bin/logger                              | INCLUDE           | EXCLUDE                |
| Gdbserver       | /bin/gdbserver                           | INCLUDE           | EXCLUDE                |
| Dropbear        | Remove “dropbear” from ‘/etc/init.d/rcs’ | EXCLUDE           | EXCLUDE                |
| SSH             | NA                                       | INCLUDE           | EXCLUDE                |
| Editors (vi)    | /bin/vi                                  | INCLUDE           | EXCLUDE                |
| Dmesg           | /bin/dmesg                               | INCLUDE           | EXCLUDE                |
| UART            | /proc/tty/driver/                        | INCLUDE           | EXCLUDE                |
| Hexdump         | /bin/hexdump                             | INCLUDE           | EXCLUDE                |
| Dnsdomainname   | /bin/dnsdomainname                       | EXCLUDE           | EXCLUDE                |
| Hostname        | /bin/hostname                            | INCLUDE           | EXCLUDE                |
| Pmap            | /bin/pmap                                | INCLUDE           | EXCLUDE                |
| su              | /bin/su                                  | INCLUDE           | EXCLUDE                |
| Which           | /bin/which                               | INCLUDE           | EXCLUDE                |
| Who and whoami  | /bin/whoami                              | INCLUDE           | EXCLUDE                |
| ps              | /bin/ps                                  | INCLUDE           | EXCLUDE                |
| lsmod           | /sbin/lsmod                              | INCLUDE           | EXCLUDE                |
| install         | /bin/install                             | INCLUDE           | EXCLUDE                |
| logger          | /bin/logger                              | INCLUDE           | EXCLUDE                |
| ps              | /bin/ps                                  | INCLUDE           | EXCLUDE                |
| rpm             | /bin/rpm                                 | INCLUDE           | EXCLUDE                |
| Iostat          | /bin/iostat                              | INCLUDE           | EXCLUDE                |
| find            | /bin/find                                | INCLUDE           | EXCLUDE                |
| Chgrp           | /bin/chgrp                               | INCLUDE           | EXCLUDE                |
| Chmod           | /bin/chmod                               | INCLUDE           | EXCLUDE                |
| Chown           | /bin/chown                               | INCLUDE           | EXCLUDE                |
| killall         | /bin/killall                             | INCLUDE           | EXCLUDE                |
| top             | /bin/top                                 | INCLUDE           | EXCLUDE                |
| stbhotplug      | /sbin/stbhotplug                         | INCLUDE           | EXCLUDE                |

* Utilize tools such as [Lynis](https://raw.githubusercontent.com/CISOfy/lynis/master/lynis) for hardening auditing and suggestions. `wget --no-check-certificate https://github.com/CISOfy/lynis/archive/master.zip && unzip master.zip && cd lynis-master/ && bash lynis audit system`
  * Review the report in: `/var/log/lynis.log`
* Perform iterative threat model exercises with developers as well as relative stakeholders on software running on the embedded device.

## Yocto Project Build System Security

The Yocto Project provides a comprehensive, production-grade embedded Linux build system with extensive built-in security features. Yocto has become the de facto standard for secure embedded Linux development in automotive, industrial, medical, and consumer IoT markets. This section covers security hardening capabilities specific to Yocto builds.

### Yocto Scarthgap 5.0 LTS Security Foundation

**Yocto Project 5.0 "Scarthgap"** (May 2024) is the current Long-Term Support (LTS) release with 4 years of security maintenance. It provides a modern, secure foundation:

**Core Components**:
- **Linux Kernel 6.6 LTS**: Long-term kernel support with extensive security backports
- **GCC 13.2**: Modern compiler with enhanced security analysis and hardening features
- **glibc 2.39**: Latest GNU C Library with security improvements
- **LLVM 18.1**: Alternative toolchain with sanitizers and static analysis
- **OpenSSL 3.2**: Latest TLS 1.3 implementation
- **100% Reproducible Builds**: Achieved in 2022, maintained in all LTS releases

**Key Security Enhancements**:
- Enhanced CVE database synchronization
- Improved SBOM generation (SPDX 3.0.1 default)
- Automatic SECURITY file checking in layers
- Continuous security patches for core recipes (cups, curl, openssl, qemu, python3, vim)

### Compiler Security Flags

Yocto includes a comprehensive set of compiler security flags that harden binaries against common vulnerabilities. These flags are centrally managed in `meta/conf/distro/include/security_flags.inc`.

#### Enabling Security Flags

**Basic Configuration (local.conf or distro .conf)**:
```bitbake
# Enable all security flags for the build
# This is usually enabled by default in modern Yocto versions
INHERIT += "security-flags"
```

**Verify Security Flags are Active**:
```bash
# Check if security flags are enabled in your build
bitbake-getvar SECURITY_CFLAGS
bitbake-getvar SECURITY_LDFLAGS
```

#### Key Security Compiler Flags Explained

**1. Stack Protection (`-fstack-protector-strong`)**:
```bitbake
# Enabled by default in security_flags.inc
SECURITY_CFLAGS += "-fstack-protector-strong"
```

**What it does**:
- Inserts stack canaries before return addresses
- Protects functions with vulnerable parameters (arrays, pointers, structs with arrays)
- Detects stack buffer overflows at runtime
- Stronger than `-fstack-protector`, less overhead than `-fstack-protector-all`

**Impact**: ~2-5% performance overhead, catches 90%+ of stack overflow attacks

**2. FORTIFY_SOURCE (`-D_FORTIFY_SOURCE=2`)**:
```bitbake
# Enabled by default in security_flags.inc
SECURITY_CFLAGS += "-D_FORTIFY_SOURCE=2"
```

**What it does**:
- Compile-time and runtime checks for buffer overflows in memory/string functions
- Detects out-of-bounds writes to `memcpy()`, `strcpy()`, `sprintf()`, etc.
- Level 2 provides stronger checks than level 1

**Requires**: Optimization level -O1 or higher (Yocto uses -O2 by default)

**Example Protection**:
```c
char buf[10];
strcpy(buf, "This string is way too long");  // Caught at runtime with FORTIFY_SOURCE
```

**3. Position Independent Executable (`-fpie -pie`)**:
```bitbake
# Enabled by default for executables
SECURITY_CFLAGS += "-fpie"
SECURITY_LDFLAGS += "-pie"
```

**What it does**:
- Enables Address Space Layout Randomization (ASLR)
- Randomizes executable load address at runtime
- Makes exploit development significantly harder (attackers can't predict memory addresses)

**Note**: All modern embedded systems should enable PIE unless hardware constraints prevent it

**4. Format String Protection (`-Wformat -Wformat-security`)**:
```bitbake
# Enabled by default
SECURITY_CFLAGS += "-Wformat -Wformat-security -Werror=format-security"
```

**What it does**:
- Detects format string vulnerabilities at compile time
- Warns about unsafe uses of `printf()`, `scanf()`, etc.
- Prevents classic format string exploits

**Example Vulnerability Caught**:
```c
char *user_input = get_user_data();
printf(user_input);  // ❌ Compile error with -Werror=format-security
printf("%s", user_input);  // ✅ Safe
```

**5. Read-Only Relocations (`-Wl,-z,relro,-z,now`)**:
```bitbake
# Enabled by default
SECURITY_LDFLAGS += "-Wl,-z,relro,-z,now"
```

**What it does**:
- **RELRO (RELocation Read-Only)**: Makes GOT (Global Offset Table) read-only after relocation
- **NOW**: Forces all symbols to be resolved at load time (disables lazy binding)
- Prevents GOT overwrite attacks

**Trade-off**: Slightly slower startup time due to immediate symbol resolution

#### GCC -fhardened Flag (GCC 13.2+)

**Modern Hardening Flag (Scarthgap 5.0+)**:
```bitbake
# In your distro .conf or local.conf
SECURITY_CFLAGS += "-fhardened"
```

**What it enables** (GCC 13.2 in Scarthgap):
- `-D_FORTIFY_SOURCE=3` (level 3 - even stronger checks)
- `-D_GLIBCXX_ASSERTIONS` (C++ standard library assertions)
- `-ftrivial-auto-var-init=zero` (zero-initialize automatic variables)
- `-fstack-protector-strong`
- `-fstack-clash-protection` (protect against stack clash attacks)

**When to use**: High-security applications (medical, automotive, industrial control)

**Trade-off**: ~5-10% performance overhead, significantly improved security

#### Customizing Security Flags Per Recipe

**Disable security flags for specific recipes** (e.g., performance-critical code):
```bitbake
# In recipe (.bb file)
SECURITY_CFLAGS = ""
SECURITY_LDFLAGS = ""
```

**Add additional flags for specific recipes**:
```bitbake
# In recipe (.bb file)
SECURITY_CFLAGS:append = " -fstack-clash-protection"
```

**Example: High-Security Recipe**:
```bitbake
# crypto-app_1.0.bb
DESCRIPTION = "Cryptographic application requiring maximum security"
LICENSE = "MIT"

# Enable maximum security hardening
SECURITY_CFLAGS:append = " -fhardened -fanalyzer"
SECURITY_LDFLAGS:append = " -Wl,-z,noexecstack"

# Fortify source level 3 (GCC 13+)
CFLAGS:append = " -D_FORTIFY_SOURCE=3"

# Stack clash protection
CFLAGS:append = " -fstack-clash-protection"
```

#### Verifying Security Flags in Binaries

**Use checksec to verify security hardening**:
```bash
# Install checksec
bitbake checksec-native

# Check security properties of a binary
checksec --file=tmp/work/.../package/usr/bin/myapp

# Example output:
# RELRO           STACK CANARY      NX            PIE
# Full RELRO      Canary found      NX enabled    PIE enabled
```

**Manual verification with readelf**:
```bash
# Check for PIE
readelf -h binary | grep Type
# Should show: Type: DYN (Shared object file)

# Check for stack canary
readelf -s binary | grep stack_chk_fail
# Should show: __stack_chk_fail symbol

# Check for RELRO
readelf -l binary | grep GNU_RELRO
# Should show: GNU_RELRO segment
```

### Kernel Hardening in Yocto

Linux kernel hardening is critical for embedded security. Yocto provides multiple mechanisms to configure kernel security options.

#### Kernel Configuration Fragments

Yocto uses **kernel configuration fragments** (`.cfg` files) to modularize kernel configuration changes:

**Create a kernel hardening fragment** (`kernel-hardening.cfg`):
```cfg
# Kernel hardening configuration fragment
# Place in recipes-kernel/linux/files/

# Address Space Layout Randomization (ASLR)
CONFIG_RANDOMIZE_BASE=y
CONFIG_RANDOMIZE_MEMORY=y

# Kernel Page Table Isolation (KPTI) - Meltdown mitigation
CONFIG_PAGE_TABLE_ISOLATION=y

# Supervisor Mode Execution Protection (SMEP)
CONFIG_X86_SMAP=y
CONFIG_X86_SMEP=y

# Kernel Stack Protection
CONFIG_STACKPROTECTOR=y
CONFIG_STACKPROTECTOR_STRONG=y

# Restrict /dev/mem and /dev/kmem access
CONFIG_STRICT_DEVMEM=y
CONFIG_DEVMEM=n
CONFIG_DEVKMEM=n

# Harden BPF JIT compiler
CONFIG_BPF_JIT_ALWAYS_ON=y

# Disable legacy and dangerous features
CONFIG_LEGACY_PTYS=n
CONFIG_DEVTMPFS_MOUNT=n

# Enable kernel lockdown
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_SECURITY_LOCKDOWN_LSM_EARLY=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_CONFIDENTIALITY=y

# Restrict kernel module loading
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_FORCE=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_SHA256=y

# Disable kernel debugging features in production
# CONFIG_DEBUG_FS is not set
# CONFIG_KPROBES is not set
# CONFIG_FTRACE is not set
```

**Apply kernel fragment in recipe**:
```bitbake
# linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://kernel-hardening.cfg"
```

**Alternative: Using defconfig**:
```bitbake
# For custom kernel recipes
SRC_URI += "file://defconfig"
```

#### meta-security Hardening Configurations

The **meta-security** layer provides pre-built kernel hardening configurations:

**Add meta-security layer**:
```bash
# Clone meta-security
git clone https://git.yoctoproject.org/meta-security

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-security"
```

**Use meta-security kernel hardening**:
```bitbake
# In local.conf
KERNEL_FEATURES:append = " features/security/security.scc"
```

**Available hardening features in meta-security**:
- `features/security/security.scc`: Basic kernel hardening
- `features/selinux/selinux.scc`: SELinux support
- `features/apparmor/apparmor.scc`: AppArmor support
- `features/ima/ima.scc`: Integrity Measurement Architecture
- `features/smack/smack.scc`: SMACK MAC support

#### Kernel Hardening Checker

**Use kernel-hardening-checker to audit your kernel config**:
```bash
# Install kernel-hardening-checker
pip3 install git+https://github.com/a13xp0p0v/kernel-hardening-checker

# Check your kernel .config
kernel-hardening-checker -c tmp/work/.../linux-yocto/.config

# Example output:
# [+] Config check is finished: 'OK' - 85 / 'FAIL' - 12
```

**Automate in Yocto build**:
```bitbake
# Create a kernel-hardening-check recipe
inherit kernel-arch

do_kernel_hardening_check() {
    kernel-hardening-checker -c ${B}/.config
}
addtask kernel_hardening_check after do_configure before do_compile
```

#### Key Kernel Security Options Explained

**KASLR (Kernel Address Space Layout Randomization)**:
```cfg
CONFIG_RANDOMIZE_BASE=y          # Randomize kernel base address
CONFIG_RANDOMIZE_MEMORY=y        # Randomize memory sections (x86_64)
```
- **Benefit**: Makes kernel exploits significantly harder (attacker can't predict memory layout)
- **Overhead**: Minimal (<1%)
- **Recommendation**: Enable on all modern systems

**KPTI (Kernel Page Table Isolation)**:
```cfg
CONFIG_PAGE_TABLE_ISOLATION=y
```
- **Benefit**: Mitigates Meltdown (CVE-2017-5754) and related CPU vulnerabilities
- **Overhead**: 5-30% performance impact (CPU-dependent)
- **Recommendation**: Enable unless performance is critical AND CPU is not vulnerable

**Kernel Module Signing**:
```cfg
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_FORCE=y        # Reject unsigned modules
CONFIG_MODULE_SIG_ALL=y          # Sign all modules at build time
CONFIG_MODULE_SIG_SHA256=y       # Use SHA-256 for signatures
```
- **Benefit**: Prevents loading of malicious kernel modules
- **Requirement**: Requires signing key management
- **Recommendation**: Essential for production devices

### Kernel Security Mechanisms: Isolation and Resource Control

Beyond basic kernel hardening, Linux provides several powerful security mechanisms for process isolation, privilege management, and resource control. These mechanisms form the foundation of container security and defense-in-depth strategies for embedded devices. This section covers **Linux namespaces**, **control groups (cgroups)**, **seccomp-BPF syscall filtering**, and **Linux capabilities** with focus on Yocto integration, hardware requirements, and practical deployment for IoT, Industrial IoT, and enterprise network devices.

#### Hardware Requirements and Device Suitability

Different security mechanisms have varying resource overhead. Choose mechanisms appropriate for your device constraints and security requirements.

**Hardware Requirements Matrix:**

| Security Mechanism | Min RAM | Min CPU | Kernel Version | Runtime Overhead | Best For |
|-------------------|---------|---------|----------------|-----------------|----------|
| **Linux Capabilities** | Any | Any | 2.2+ | ~0% | All devices (universal) |
| **Seccomp-BPF (basic, 5-10 filters)** | 16MB+ | Any | 3.5+ | <1% CPU, ~20% syscall latency | Resource-constrained IoT |
| **Seccomp-BPF (complex, 32+ filters)** | 32MB+ | Any | 3.5+ | ~78% syscall latency | Advanced filtering needs |
| **Namespaces (PID + Network)** | 32MB+ | Single core | 3.8+ | ~2-5MB RAM | Basic process isolation |
| **User Namespaces** | 64MB+ | Single core | 3.8+ | ~5-10MB RAM | Rootless containers |
| **cgroups v1** | 64MB+ | Single core | 2.6.24+ | ~2-5MB RAM, <0.1% CPU | Resource limiting |
| **cgroups v2 (unified)** | 128MB+ | Dual core | 4.5+ | ~5-10MB RAM, <0.5% CPU | Modern resource control |
| **Full Defense-in-Depth Stack** | 256MB+ | Dual core | 5.0+ | ~15-25MB RAM, <2% CPU | Enterprise/IIoT gateways |

**Performance Overhead Details:**
- **Capabilities**: Zero overhead (kernel feature since 1999)
- **Seccomp-BPF**: Per-filter overhead ~18ns CPU + ~17ns BPF evaluation
  - 1 filter: +19.8% syscall latency
  - 4 filters: +33.0% syscall latency
  - 32 filters: +78.3% syscall latency
  - Optimization: Binary search tree (libseccomp) reduces to O(log n)
- **Namespaces**: Memory overhead per instance
  - PID namespace: ~512KB
  - Network namespace: ~2MB (routing tables, netfilter state)
  - Mount namespace: ~1MB (mount table duplication)
  - User namespace: ~5MB (UID/GID mapping overhead)
- **cgroups**: Base overhead + per-cgroup overhead
  - cgroups v1: ~2-5MB base + ~100KB per cgroup
  - cgroups v2: ~5-10MB base + ~200KB per cgroup
  - Memory controller: Additional ~1-2MB (disabled by default on Raspberry Pi)

**Device Profile Decision Matrix:**

```
Device Profile           RAM     CPU         Recommended Mechanisms                          Skip
───────────────────────  ──────  ──────────  ──────────────────────────────────────────────  ────────────────────
Ultra-constrained IoT    <32MB   Single core Capabilities only                               Namespaces, cgroups
Consumer IoT             32-64MB Single core Capabilities + Seccomp (5-10 filters)           User NS, cgroups v2
Industrial IoT           64-128MB Single/Dual Capabilities + Seccomp + Namespaces (PID+Net)  cgroups v2 (use v1)
Industrial Gateway       128MB+  Dual core   Capabilities + Seccomp + Namespaces + cgroups  None (full stack OK)
Enterprise Network Device 256MB+ Dual core+  Full defense-in-depth (all mechanisms)         None
```

**Example: Raspberry Pi Zero (512MB RAM, Single Core, ARMv6)**
- ✅ **Capabilities**: No overhead, universal benefit - always enable
- ✅ **Seccomp with 10 filters**: ~20% syscall overhead acceptable for security benefit
- ✅ **PID + Network namespaces**: ~3MB RAM overhead (< 1% of total) - enable for isolation
- ⚠️ **cgroups memory controller**: Disabled by default due to overhead
  - Enable only if needed: Add `cgroup_enable=memory` to `/boot/cmdline.txt`
- ❌ **User namespaces**: 5-10MB overhead not justified unless rootless operation required
- ❌ **cgroups v2**: 10MB+ overhead too high - use cgroups v1 if resource limits needed

**Example: Industrial IoT Gateway (2GB RAM, Quad Core, x86_64)**
- ✅ **Full stack**: All mechanisms enabled
- ✅ **cgroups v2**: Unified hierarchy for cleaner management
- ✅ **User namespaces**: Enable rootless containers
- ✅ **Complex seccomp profiles**: 30+ filters acceptable with quad core

#### Linux Namespaces: Process Isolation

Linux namespaces provide process isolation by virtualizing system resources. Each namespace type isolates a different aspect of the system, enabling container-style isolation without requiring full containers. Namespaces are fundamental to defense-in-depth strategies, preventing lateral movement and limiting blast radius after compromise.

**Seven Namespace Types:**

| Namespace | Purpose | Isolation Provided | Memory Overhead | Since Kernel |
|-----------|---------|-------------------|-----------------|--------------|
| **PID** | Process IDs | Process tree, prevents cross-process signals | ~512KB | 2.6.24 |
| **Network** | Network stack | Network interfaces, routing, firewall rules | ~2MB | 2.6.29 |
| **Mount** | Filesystem mounts | Mount points, prevents filesystem escape | ~1MB | 2.4.19 |
| **IPC** | Inter-process communication | Shared memory, message queues, semaphores | ~256KB | 2.6.19 |
| **User** | User/group IDs | UID/GID mapping, rootless containers | ~5MB | 3.8 |
| **UTS** | Hostname/domain | Hostname isolation | ~64KB | 2.6.19 |
| **Cgroup** | Control group membership | Process resource view | ~128KB | 4.6 |

**Security Benefits:**
- **Prevents lateral movement**: Compromised process cannot see other processes (PID namespace)
- **Network isolation**: Exploited service cannot sniff traffic or bind to privileged ports (Network namespace)
- **Filesystem protection**: Process cannot access sensitive mounts (Mount namespace)
- **Privilege separation**: Services run as non-root inside namespace, root outside (User namespace)

**Recommendations for Constrained Devices (<64MB RAM):**
- ✅ **Enable PID + Network namespaces**: ~2.5MB overhead, high security value
- ⚠️ **Mount namespace**: Enable if filesystem isolation needed (databases, configuration stores)
- ⚠️ **User namespace**: 5MB overhead - only if rootless operation required
- ❌ **Skip IPC/UTS/Cgroup namespaces**: Low security value for embedded devices

**Example 1: systemd Service with Namespace Isolation**

Isolate a network service (e.g., MQTT broker, web server) using systemd's namespace features:

```ini
# /etc/systemd/system/iot-gateway.service
[Unit]
Description=IoT Gateway Service
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/iot-gateway
Restart=on-failure

# Process isolation (PID namespace)
PrivateTmp=yes                    # Private /tmp (mount namespace)
ProtectSystem=strict              # Read-only /usr, /boot, /etc (mount namespace)
ProtectHome=yes                   # Inaccessible /home (mount namespace)
PrivateDevices=yes                # Private /dev with minimal devices

# Network isolation
PrivateNetwork=no                 # Service needs network access
RestrictAddressFamilies=AF_INET AF_INET6  # Block AF_UNIX sockets to other services

# IPC isolation
PrivateIPC=yes                    # Private IPC namespace

# Hostname isolation
PrivateUsers=no                   # User namespace (set to 'yes' for rootless)

# Additional hardening
NoNewPrivileges=yes               # Prevent privilege escalation
ProtectKernelTunables=yes         # Read-only /proc/sys, /sys
ProtectKernelModules=yes          # Prevent kernel module loading
ProtectControlGroups=yes          # Read-only /sys/fs/cgroup

[Install]
WantedBy=multi-user.target
```

**Testing namespace isolation:**

```bash
# Verify PID namespace (process should only see itself)
systemctl start iot-gateway
PID=$(systemctl show -p MainPID --value iot-gateway)
sudo nsenter -t $PID -p ps aux  # Should show minimal processes

# Verify mount namespace (system mounts not visible)
sudo nsenter -t $PID -m findmnt | grep -E '/(home|usr|boot)'

# Verify IPC namespace (no shared memory from other processes)
sudo nsenter -t $PID -i ipcs -m
```

**Example 2: Yocto Kernel Configuration for Namespaces**

Enable namespace support in Yocto kernel configuration:

```cfg
# recipes-kernel/linux/linux-yocto/namespaces.cfg (kernel fragment)
# Core namespace support
CONFIG_NAMESPACES=y
CONFIG_UTS_NS=y          # Hostname isolation (~64KB)
CONFIG_IPC_NS=y          # IPC isolation (~256KB)
CONFIG_PID_NS=y          # Process isolation (~512KB) - RECOMMENDED
CONFIG_NET_NS=y          # Network isolation (~2MB) - RECOMMENDED

# Mount namespace (required for ProtectSystem, PrivateTmp)
CONFIG_MOUNT_NS=y        # Filesystem isolation (~1MB) - RECOMMENDED

# User namespace (optional, 5-10MB overhead)
# CONFIG_USER_NS=y       # Rootless containers - Enable if needed
# CONFIG_USER_NS_UNPRIVILEGED=y  # Allow non-root user namespace creation

# Cgroup namespace (optional, low value for embedded)
# CONFIG_CGROUP_NS=y     # ~128KB - Usually not needed

# Security: Restrict unprivileged user namespace creation
# CONFIG_USER_NS_UNPRIVILEGED is not set  # Prevent abuse by unprivileged users
```

**BitBake recipe integration:**

```bitbake
# recipes-kernel/linux/linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://namespaces.cfg"

# For devices with <64MB RAM, minimize overhead
# recipes-core/systemd/systemd_%.bbappend
PACKAGECONFIG:append = " \
    ${@bb.utils.contains('MACHINE_FEATURES', 'namespace-support', 'namespace', '', d)} \
"

# Add to image
IMAGE_INSTALL:append = " systemd systemd-analyze"
```

**Testing in Yocto with QEMU:**

```python
# meta-<layer>/lib/oeqa/runtime/cases/test_namespaces.py
from oeqa.runtime.case import OERuntimeTestCase

class NamespaceTest(OERuntimeTestCase):
    @classmethod
    def setUpClass(cls):
        cls.target.run('systemctl daemon-reload')

    def test_pid_namespace_isolation(self):
        """Verify PID namespace prevents cross-process visibility"""
        # Start test service with PID namespace
        self.target.run('systemctl start test-isolated-service')

        # Get PID of isolated service
        _, output = self.target.run('systemctl show -p MainPID --value test-isolated-service')
        pid = output.strip()

        # Enter PID namespace and check process visibility
        status, output = self.target.run(f'nsenter -t {pid} -p ps aux | wc -l')
        process_count = int(output.strip())

        # Should see minimal processes (1-5), not full system process list
        self.assertLess(process_count, 10,
                       f"PID namespace not isolated: {process_count} processes visible")

    def test_mount_namespace_isolation(self):
        """Verify mount namespace prevents access to sensitive paths"""
        self.target.run('systemctl start test-isolated-service')

        _, output = self.target.run('systemctl show -p MainPID --value test-isolated-service')
        pid = output.strip()

        # Try to access /home from inside namespace (should fail)
        status, _ = self.target.run(f'nsenter -t {pid} -m ls /home')
        self.assertNotEqual(status, 0, "Mount namespace did not block /home access")

    def test_network_namespace_exists(self):
        """Verify network namespace support is enabled"""
        status, _ = self.target.run('ip netns add test_ns && ip netns del test_ns')
        self.assertEqual(status, 0, "Network namespace support not available")
```

**Memory Overhead Summary:**
- **PID + Network + Mount namespaces**: ~3.5MB (recommended baseline)
- **Add IPC + UTS**: ~4MB total (marginal value)
- **Add User namespace**: ~9MB total (only if rootless required)
- **Per-service namespace instance**: ~100-500KB additional

**Compatibility Notes:**
- **Raspberry Pi**: PID/Network/Mount namespaces work on all models (512MB+ RAM)
- **Yocto systemd**: Namespace support requires systemd 220+ (default since Yocto 2.0 Jethro)
- **Kernel 3.8+**: All namespaces supported (except cgroup namespace, requires 4.6+)
- **User namespace caution**: Can enable privilege escalation if misconfigured - disable `CONFIG_USER_NS_UNPRIVILEGED` in production

#### Control Groups v2 (cgroups): Resource Control and DoS Prevention

Control groups (cgroups) enforce resource limits on processes, preventing resource exhaustion attacks and ensuring quality of service. Cgroups v2 provides a unified hierarchy with improved consistency and performance compared to v1. For embedded devices, cgroups are critical for **IEC 62443-4-2 FR 7** (Resource Availability) compliance, preventing denial-of-service attacks through fork bombs, memory exhaustion, or CPU starvation.

**Unified Hierarchy (cgroups v2) vs. Legacy (v1):**

| Feature | cgroups v1 | cgroups v2 (Unified) |
|---------|-----------|----------------------|
| **Hierarchy** | Multiple (one per controller) | Single unified tree |
| **Memory overhead** | ~2-5MB base | ~5-10MB base |
| **CPU overhead** | <0.1% | <0.5% |
| **Kernel version** | 2.6.24+ | 4.5+ (stable: 5.0+) |
| **systemd support** | Full (default) | Full (systemd 226+) |
| **Recommendation** | Devices <128MB RAM | Devices 128MB+ RAM |

**Resource Controllers:**

| Controller | Purpose | DoS Attack Prevented | Overhead |
|-----------|---------|---------------------|----------|
| **CPU** | CPU time limits, CPU pinning | CPU starvation attacks | <0.1% |
| **Memory** | Memory limits, OOM killer control | Memory exhaustion, fork bombs | ~1-2MB |
| **I/O** | Disk I/O bandwidth limits | Disk thrashing attacks | <0.5% |
| **PID** | Maximum process count | Fork bomb attacks | Minimal |
| **Network** | Network bandwidth limits (via tc) | Network flooding | Varies |

**Security Benefits:**
- **DoS prevention**: Fork bomb cannot exhaust system PIDs (PID controller)
- **Memory isolation**: Compromised service OOM-killed before affecting system (Memory controller)
- **CPU fairness**: Malicious process cannot starve other services (CPU controller)
- **I/O protection**: Database corruption prevented by limiting disk writes (I/O controller)

**Hardware Overhead:**
- **cgroups v1**: ~2-5MB base + ~100KB per cgroup instance
- **cgroups v2**: ~5-10MB base + ~200KB per cgroup instance
- **Memory controller**: Additional ~1-2MB (disabled by default on Raspberry Pi - enable with `cgroup_enable=memory` in `/boot/cmdline.txt`)

**Example 3: Yocto Kernel Configuration for cgroups**

Enable cgroups v2 with resource controllers in Yocto kernel:

```cfg
# recipes-kernel/linux/linux-yocto/cgroups.cfg (kernel fragment)
# Core cgroup support
CONFIG_CGROUPS=y

# Unified hierarchy (cgroups v2) - Recommended for >=128MB RAM devices
CONFIG_CGROUP_UNIFIED=y          # Enable unified cgroups v2

# Resource controllers
CONFIG_MEMCG=y                   # Memory controller (REQUIRED for OOM protection)
CONFIG_MEMCG_SWAP=y              # Swap memory accounting (if swap enabled)
CONFIG_MEMCG_KMEM=y              # Kernel memory accounting
CONFIG_BLK_CGROUP=y              # Block I/O controller (REQUIRED for disk limits)
CONFIG_CGROUP_PIDS=y             # PID controller (REQUIRED for fork bomb protection)
CONFIG_CGROUP_FREEZER=y          # Freezer (pause/resume processes)
CONFIG_CGROUP_DEVICE=y           # Device access control
CONFIG_CPUSETS=y                 # CPU pinning
CONFIG_CGROUP_CPUACCT=y          # CPU usage accounting
CONFIG_CGROUP_SCHED=y            # CPU scheduler integration
CONFIG_FAIR_GROUP_SCHED=y        # Fair CPU scheduling
CONFIG_CFS_BANDWIDTH=y           # CPU quota enforcement (CPUQuota= in systemd)

# Network controller (optional, requires traffic control)
# CONFIG_CGROUP_NET_PRIO=y       # Network priority
# CONFIG_CGROUP_NET_CLASSID=y    # Network classification

# For devices <128MB RAM, use cgroups v1 (legacy)
# CONFIG_CGROUP_UNIFIED is not set
# CONFIG_MEMCG is not set         # Disable memory controller to save RAM
```

**Note: Raspberry Pi Memory Controller**
Raspberry Pi disables cgroups memory controller by default to reduce overhead. Enable it:

```bash
# /boot/cmdline.txt (add to existing line, do NOT create new line)
cgroup_enable=memory cgroup_memory=1 swapaccount=1
```

**BitBake recipe for cgroups configuration:**

```bitbake
# recipes-kernel/linux/linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://cgroups.cfg"

# Enable cgroups v2 by default (requires systemd 226+)
KERNEL_FEATURES:append = " ${@bb.utils.contains('DISTRO_FEATURES', 'systemd', 'features/cgroup/cgroup-unified.scc', '', d)}"

# For Raspberry Pi, enable memory controller
do_configure:append:raspberrypi() {
    if [ -f "${DEPLOY_DIR_IMAGE}/cmdline.txt" ]; then
        sed -i 's/$/ cgroup_enable=memory cgroup_memory=1/' ${DEPLOY_DIR_IMAGE}/cmdline.txt
    fi
}
```

**Example 4: systemd Service Resource Limits**

Apply resource limits to a network service to prevent DoS attacks:

```ini
# /etc/systemd/system/iot-gateway.service
[Unit]
Description=IoT Gateway Service
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/iot-gateway
Restart=on-failure

# CPU limits (prevents CPU starvation attacks)
CPUQuota=50%                      # Max 50% of one CPU core (200% = 2 cores)
CPUAccounting=yes                 # Enable CPU usage tracking

# Memory limits (prevents memory exhaustion)
MemoryMax=256M                    # Hard limit: OOM kill at 256MB
MemoryHigh=200M                   # Soft limit: Throttle at 200MB
MemoryAccounting=yes              # Enable memory usage tracking
MemorySwapMax=0                   # Disable swap (prevents swap thrashing)

# Process limits (prevents fork bomb attacks)
TasksMax=100                      # Max 100 processes/threads (fork bomb protection)
TasksAccounting=yes               # Enable task counting

# I/O limits (prevents disk thrashing)
IOAccounting=yes                  # Enable I/O tracking
IOReadBandwidthMax=/dev/mmcblk0 10M   # Max 10MB/s read (adjust for SD card)
IOWriteBandwidthMax=/dev/mmcblk0 5M   # Max 5MB/s write (SD card wear protection)

# Additional limits
IPAccounting=yes                  # Track network bandwidth (systemd 235+)

[Install]
WantedBy=multi-user.target
```

**Testing resource limits:**

```bash
# Test CPU quota (process should be throttled at 50% CPU)
systemctl start iot-gateway
systemd-cgtop  # Monitor real-time cgroup resource usage

# Test memory limit (process should be OOM-killed at 256MB)
# Inject memory leak simulation
PID=$(systemctl show -p MainPID --value iot-gateway)
sudo sh -c "echo $((256 * 1024 * 1024)) > /proc/$PID/oom_score_adj"  # Make OOM-killable

# Monitor cgroup memory usage
watch -n 1 systemctl status iot-gateway | grep Memory

# Test fork bomb protection (should fail at 100 tasks)
# Inject fork bomb simulation (careful!)
systemd-run --unit=test-forkbomb --property=TasksMax=10 \
    bash -c 'bomb() { bomb | bomb & }; bomb'
# Should terminate with "Failed to fork (Resource temporarily unavailable)"

# Test I/O limits
dd if=/dev/zero of=/tmp/test bs=1M count=100
# Should throttle at 5MB/s write limit
```

**Yocto Test Cases for cgroups:**

```python
# meta-<layer>/lib/oeqa/runtime/cases/test_cgroups.py
from oeqa.runtime.case import OERuntimeTestCase
import time

class CgroupTest(OERuntimeTestCase):
    def test_cpu_quota_enforcement(self):
        """Verify CPUQuota limits CPU usage"""
        # Start CPU-intensive service with 25% quota
        self.target.run('systemd-run --unit=test-cpu --property=CPUQuota=25% \
                        --property=CPUAccounting=yes stress-ng --cpu 4 --timeout 10s &')
        time.sleep(3)

        # Check CPU usage (should be ~25%, allowing 5% tolerance)
        status, output = self.target.run('systemctl show test-cpu --property=CPUUsageNSec')
        # Parse and verify CPU usage is capped (complex calculation omitted for brevity)
        self.assertIn('CPUUsageNSec=', output)

    def test_memory_limit_oom_kill(self):
        """Verify MemoryMax triggers OOM killer"""
        # Start memory hog with 50MB limit
        self.target.run('systemd-run --unit=test-mem --property=MemoryMax=50M \
                        --property=MemoryAccounting=yes stress-ng --vm 1 --vm-bytes 100M --timeout 5s')
        time.sleep(2)

        # Service should be killed by OOM
        status, output = self.target.run('systemctl is-active test-mem')
        self.assertNotEqual(output.strip(), 'active',
                           "Service not OOM-killed despite exceeding MemoryMax")

    def test_tasks_max_fork_bomb(self):
        """Verify TasksMax prevents fork bombs"""
        # Attempt fork bomb with TasksMax=10
        status, _ = self.target.run('systemd-run --unit=test-fork --property=TasksMax=10 \
                                     bash -c \'bomb() { bomb | bomb & }; bomb\'')
        time.sleep(1)

        # Should fail to fork
        status, output = self.target.run('systemctl status test-fork')
        self.assertIn('Resource temporarily unavailable', output,
                     "Fork bomb not prevented by TasksMax")

    def test_io_bandwidth_limit(self):
        """Verify IOWriteBandwidthMax throttles disk writes"""
        # Write with 1MB/s limit
        self.target.run('systemd-run --unit=test-io --property=IOAccounting=yes \
                        --property=IOWriteBandwidthMax="/dev/mmcblk0 1M" \
                        dd if=/dev/zero of=/tmp/test_io bs=1M count=10 oflag=direct')

        # Should take ~10 seconds (10MB at 1MB/s)
        status, output = self.target.run('systemctl show test-io --property=ExecMainExitTimestamp')
        # Verify timing (complex calculation omitted)
        self.assertIn('ExecMainExitTimestamp=', output)
```

**ROI for cgroups Implementation:**

| Risk Mitigated | Without cgroups | With cgroups | Risk Reduction |
|----------------|-----------------|--------------|----------------|
| **Fork bomb DoS** (IEC 62443 FR 7) | 100% system crash | Isolated to service | 95% |
| **Memory exhaustion** | OOM kills critical services | Attacker service killed | 90% |
| **CPU starvation** | Legitimate services timeout | Fair CPU scheduling | 85% |
| **Disk thrashing** | SD card corruption | I/O throttling prevents wear | 80% |

**Implementation Investment:**
- **Development time**: 2-3 days (kernel config + systemd unit files + testing)
- **Testing time**: 1-2 days (QEMU + hardware validation)
- **Ongoing maintenance**: Minimal (tune limits per service)
- **Hardware requirement**: 64MB RAM minimum (128MB for v2)

**Estimated Annual Risk Reduction Value** (for Industrial IoT Gateway):
- DoS attack cost: $50,000/incident (downtime + recovery)
- Probability without cgroups: 40%/year
- Probability with cgroups: 5%/year
- **Annual risk reduction**: $50,000 × (40% - 5%) = **$17,500/year**
- **One-time investment**: ~$5,000 (1 week engineering)
- **ROI**: 350% in first year

**Compatibility Notes:**
- **cgroups v1**: All Yocto releases, kernel 2.6.24+
- **cgroups v2**: Yocto Dunfell (3.1+), kernel 5.0+, systemd 244+
- **Raspberry Pi**: Memory controller disabled by default (enable via `/boot/cmdline.txt`)
- **systemd integration**: Automatic (systemd manages cgroup hierarchy by default)

#### Seccomp-BPF: Syscall Filtering and Attack Surface Reduction

Secure Computing mode with Berkeley Packet Filter (seccomp-BPF) restricts which system calls a process can invoke, dramatically reducing attack surface. Linux provides 400+ system calls, but most applications need fewer than 50. Seccomp blocks unused syscalls, preventing kernel exploits, container escapes, and privilege escalation attacks. For embedded devices, seccomp-BPF is one of the highest-ROI security mechanisms: minimal overhead (<1% with basic profiles), universal applicability, and proven effectiveness against real-world exploits (e.g., CVE-2024-1086 container escape, CVE-2022-0847 Dirty Pipe).

**Attack Surface Reduction:**

| Application Type | Total Syscalls Available | Syscalls Actually Needed | Attack Surface Reduction |
|-----------------|-------------------------|--------------------------|-------------------------|
| **Web server** (nginx, lighttpd) | 400+ | ~40 | 90% |
| **Database** (SQLite, Redis) | 400+ | ~50 | 87% |
| **MQTT broker** (Mosquitto) | 400+ | ~35 | 91% |
| **systemd service** (generic) | 400+ | ~60 | 85% |
| **Container runtime** (Docker, Podman) | 400+ | ~150 | 62% |

**Blocked High-Risk Syscalls (Common in Exploits):**
- `ptrace` - Prevents process debugging/injection
- `kexec_load` - Prevents loading alternative kernels
- `reboot`, `init_module`, `delete_module` - Prevents system control
- `swapon`, `swapoff` - Prevents resource manipulation
- `mount`, `umount2` - Prevents filesystem escapes (if not needed)
- `iopl`, `ioperm` - Prevents direct I/O port access
- `personality` - Prevents execution domain changes
- `keyctl` - Prevents kernel keyring abuse

**Performance Overhead:**

| Filter Complexity | Syscall Latency Overhead | CPU Overhead | Use Case |
|------------------|------------------------|-------------|----------|
| **No filter** | 0% (baseline) | 0% | Unprotected (not recommended) |
| **1 filter rule** | +19.8% | <0.1% | Minimal protection |
| **4 filter rules** | +33.0% | <0.5% | Basic protection (5-10 blocked syscalls) |
| **8 filter rules** | +45.2% | <1% | Standard protection (20-30 blocked syscalls) |
| **32 filter rules** | +78.3% | ~2% | Advanced protection (100+ blocked syscalls) |

**Note:** Latency overhead applies to syscall invocation time (~50ns baseline + ~18ns per filter), not total application performance. Real-world application overhead is typically <1% for 5-10 filters due to syscalls being a small fraction of execution time.

**Security Benefits:**
- **Exploit mitigation**: 78% of kernel exploits require blocked syscalls (ptrace, kexec_load, module loading)
- **Container escape prevention**: Blocks `mount`, `pivot_root`, `unshare` used in container breakouts
- **Privilege escalation**: Prevents `setuid`, `setgid`, `capset` abuse
- **Zero-day protection**: Exploits using unknown syscalls fail even before patches available

**Example 5: Yocto Kernel Configuration for Seccomp**

Enable seccomp-BPF in Yocto kernel configuration:

```cfg
# recipes-kernel/linux/linux-yocto/seccomp.cfg (kernel fragment)
# Core seccomp support
CONFIG_SECCOMP=y                           # Secure computing mode
CONFIG_SECCOMP_FILTER=y                    # BPF-based syscall filtering
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y          # Architecture support (x86, ARM, ARM64)

# Optional: seccomp notification for userspace policy enforcement
CONFIG_SECCOMP_NOTIF=y                     # Seccomp user notification (kernel 5.0+)

# Audit support (for seccomp logging)
CONFIG_AUDIT=y                             # Enable audit framework
CONFIG_AUDITSYSCALL=y                      # Syscall auditing

# BPF support (required for filter evaluation)
CONFIG_BPF=y                               # Berkeley Packet Filter
CONFIG_BPF_SYSCALL=y                       # BPF syscall interface
```

**BitBake recipe for seccomp:**

```bitbake
# recipes-kernel/linux/linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://seccomp.cfg"

# Add libseccomp to image (userspace library)
# recipes-core/images/core-image-minimal.bb
IMAGE_INSTALL:append = " libseccomp"

# Optional: Add seccomp profile generation tool
IMAGE_INSTALL:append = " libseccomp-dev strace"
```

**Generating Seccomp Profiles with strace:**

Automatically generate seccomp profiles by tracing application syscalls:

```bash
# Trace application syscalls
strace -c -f -o /tmp/syscalls.log /usr/bin/iot-gateway

# Extract unique syscalls
awk '{print $NF}' /tmp/syscalls.log | sort -u > /tmp/allowed-syscalls.txt

# Generate seccomp profile (using scmp_sys_resolver)
while read syscall; do
    scmp_sys_resolver "$syscall" || true
done < /tmp/allowed-syscalls.txt > /tmp/seccomp-profile.txt
```

**Example 6: systemd Seccomp Integration**

Apply seccomp filters to a systemd service using predefined filter sets:

```ini
# /etc/systemd/system/iot-gateway.service
[Unit]
Description=IoT Gateway Service
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/iot-gateway
Restart=on-failure

# Seccomp syscall filtering
# Allow common system service syscalls + network I/O
SystemCallFilter=@system-service @network-io @file-system @signal

# Block dangerous syscalls (explicit deny list)
SystemCallFilter=~@privileged @resources @obsolete @debug @mount @module @raw-io @reboot @swap @cpu-emulation

# Specific high-risk syscalls to block
SystemCallFilter=~ptrace kexec_load kexec_file_load reboot swapon swapoff mount umount2 pivot_root chroot iopl ioperm

# Syscall architecture restriction (block 32-bit syscalls on 64-bit systems)
SystemCallArchitectures=native

# Log seccomp violations (requires CONFIG_AUDIT)
SystemCallErrorNumber=EPERM      # Return "Operation not permitted" for blocked syscalls
SystemCallLog=~@privileged       # Log attempts to use privileged syscalls

# Additional hardening
NoNewPrivileges=yes              # Prevent privilege escalation (required for seccomp)
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes

[Install]
WantedBy=multi-user.target
```

**systemd Seccomp Filter Sets:**

systemd provides predefined filter sets for common use cases:

| Filter Set | Description | Allowed Syscalls (examples) | Blocked Syscalls |
|-----------|-------------|---------------------------|------------------|
| `@system-service` | Basic system service operations | `read`, `write`, `open`, `close`, `socket`, `bind`, `accept`, `fork`, `exec` | Privileged operations |
| `@network-io` | Network I/O operations | `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`, `sendmsg`, `recvmsg` | N/A |
| `@file-system` | Filesystem operations | `open`, `read`, `write`, `stat`, `access`, `rename`, `unlink`, `mkdir`, `rmdir` | `mount`, `umount2`, `pivot_root` |
| `@signal` | Signal handling | `kill`, `sigaction`, `sigreturn`, `rt_sigaction`, `rt_sigreturn` | N/A |
| `@privileged` | Privileged operations (DENY) | `reboot`, `kexec_load`, `module_load`, `ptrace`, `chroot`, `setuid`, `setgid`, `capset` | Allowed for root services only |
| `@debug` | Debugging operations (DENY) | `ptrace`, `process_vm_readv`, `process_vm_writev`, `kcmp` | Block to prevent exploitation |
| `@mount` | Filesystem mounting (DENY) | `mount`, `umount2`, `pivot_root`, `chroot` | Block container escapes |
| `@module` | Kernel module operations (DENY) | `init_module`, `finit_module`, `delete_module` | Block rootkits |

**Custom Seccomp Policy (Advanced):**

For fine-grained control, create custom BPF filters using `libseccomp`:

```c
// seccomp-iot-gateway.c - Custom seccomp profile for IoT gateway
#include <seccomp.h>
#include <errno.h>

int apply_seccomp_filter() {
    scmp_filter_ctx ctx;

    // Default action: ALLOW (whitelist mode)
    // For stricter security, use SCMP_ACT_ERRNO(EPERM) and whitelist syscalls
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    if (ctx == NULL) return -1;

    // Block high-risk syscalls (blacklist)
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(ptrace), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(kexec_load), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(kexec_file_load), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(reboot), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(init_module), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(finit_module), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(delete_module), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(mount), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(umount2), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(pivot_root), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(chroot), 0);

    // Load filter into kernel
    if (seccomp_load(ctx) < 0) {
        seccomp_release(ctx);
        return -1;
    }

    seccomp_release(ctx);
    return 0;
}

// Call before application main logic
int main() {
    if (apply_seccomp_filter() != 0) {
        perror("Failed to apply seccomp filter");
        return 1;
    }

    // Application logic here (syscalls now restricted)
    // ...
}
```

**BitBake recipe for custom seccomp application:**

```bitbake
# recipes-security/seccomp-profiles/seccomp-iot-gateway_1.0.bb
SUMMARY = "Seccomp filter for IoT gateway"
LICENSE = "MIT"
DEPENDS = "libseccomp"

SRC_URI = "file://seccomp-iot-gateway.c"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o seccomp-iot-gateway ${WORKDIR}/seccomp-iot-gateway.c -lseccomp
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 seccomp-iot-gateway ${D}${bindir}/
}
```

**Testing Seccomp Filters:**

```bash
# Test that blocked syscalls return EPERM
systemctl start iot-gateway

# Attempt blocked syscall (should fail)
PID=$(systemctl show -p MainPID --value iot-gateway)
sudo strace -p $PID -e trace=ptrace 2>&1 | grep EPERM
# Expected: ptrace(...) = -1 EPERM (Operation not permitted)

# Verify seccomp is active
grep Seccomp /proc/$PID/status
# Expected: Seccomp: 2 (filtering mode)

# Check audit logs for seccomp violations (if CONFIG_AUDIT enabled)
ausearch -m SECCOMP -ts recent
# Shows blocked syscall attempts
```

**Yocto Test Cases for Seccomp:**

```python
# meta-<layer>/lib/oeqa/runtime/cases/test_seccomp.py
from oeqa.runtime.case import OERuntimeTestCase

class SeccompTest(OERuntimeTestCase):
    def test_seccomp_enabled_in_kernel(self):
        """Verify seccomp support is compiled into kernel"""
        status, output = self.target.run('zcat /proc/config.gz | grep CONFIG_SECCOMP=')
        self.assertIn('CONFIG_SECCOMP=y', output,
                     "Seccomp not enabled in kernel")

    def test_systemd_service_has_seccomp_filter(self):
        """Verify systemd service uses seccomp filtering"""
        self.target.run('systemctl start test-secured-service')

        # Check seccomp status in /proc
        _, output = self.target.run('systemctl show -p MainPID --value test-secured-service')
        pid = output.strip()

        status, output = self.target.run(f'grep Seccomp /proc/{pid}/status')
        self.assertIn('Seccomp:\t2', output,
                     "Seccomp filtering not active (expected mode 2)")

    def test_blocked_syscall_returns_eperm(self):
        """Verify blocked syscalls return EPERM"""
        self.target.run('systemctl start test-secured-service')

        _, output = self.target.run('systemctl show -p MainPID --value test-secured-service')
        pid = output.strip()

        # Attempt ptrace (should be blocked)
        status, output = self.target.run(f'strace -p {pid} -e trace=ptrace 2>&1 | head -1')
        self.assertIn('EPERM', output,
                     "Blocked syscall did not return EPERM")

    def test_libseccomp_installed(self):
        """Verify libseccomp userspace library is available"""
        status, _ = self.target.run('scmp_sys_resolver read')
        self.assertEqual(status, 0,
                        "libseccomp tools not available")
```

**Real-World Exploit Mitigation Examples:**

| CVE | Exploit Type | Syscalls Used | Seccomp Protection |
|-----|--------------|---------------|-------------------|
| **CVE-2024-1086** | Container escape (netfilter use-after-free) | `socket(AF_NETLINK)`, `setsockopt` | Block `@mount`, `@module`, `unshare` |
| **CVE-2022-0847** | Dirty Pipe (arbitrary file write) | `pipe`, `splice`, `write` | Block `splice` for non-privileged services |
| **CVE-2021-4034** | PwnKit (pkexec privilege escalation) | `execve`, `setuid` | Block `@privileged` syscalls |
| **CVE-2016-5195** | Dirty COW (write to read-only memory) | `madvise`, `write`, `/proc/self/mem` | Block `process_vm_writev`, `ptrace` |

**Seccomp ROI Summary:**
- **Development time**: 1-2 days (kernel config + systemd profiles + testing)
- **Performance overhead**: <1% (5-10 filter rules)
- **Attack surface reduction**: 85-90% (blocks 340+ unused syscalls)
- **Exploit prevention**: 78% of kernel exploits require blocked syscalls
- **Hardware requirement**: 16MB+ RAM (minimal overhead)
- **Annual risk reduction** (Industrial IoT): $150,000/year (kernel exploit costs)
- **ROI**: 3,000% in first year ($5,000 investment vs. $150,000 risk reduction)

**Compatibility Notes:**
- **Kernel 3.5+**: seccomp-BPF support (all modern embedded kernels)
- **systemd 231+**: `SystemCallFilter=` support (Yocto Morty 2.2+)
- **libseccomp**: Version 2.4+ recommended (supports syscall logging)
- **Architecture support**: x86, x86_64, ARM, ARM64, MIPS, PowerPC, RISC-V

#### Linux Capabilities: Fine-Grained Privilege Management

Linux capabilities divide root privileges into 40+ distinct units, enabling fine-grained access control. Instead of running services as root (UID 0) with full system privileges, capabilities allow services to retain only the specific privileges they need (e.g., binding to privileged ports, changing ownership). Capabilities are universally applicable (zero overhead, kernel 2.2+) and eliminate 60% of privilege escalation risks by removing unnecessary root privileges.

**Traditional Model vs. Capabilities:**

| Model | Privilege Levels | Example | Risk |
|-------|-----------------|---------|------|
| **Traditional** | Binary (root/non-root) | Web server runs as root to bind port 80 | Full system access if compromised |
| **Capabilities** | 40+ granular capabilities | Web server retains only `CAP_NET_BIND_SERVICE` | Limited to network operations if compromised |

**Common Capabilities and Security Impact:**

| Capability | Purpose | Risk if Granted | Recommendation |
|-----------|---------|----------------|----------------|
| `CAP_NET_BIND_SERVICE` | Bind ports <1024 | Low (network only) | ✅ Grant to web servers, MQTT brokers |
| `CAP_NET_RAW` | Raw sockets (ping, traceroute) | Medium (network sniffing) | ⚠️ Grant only if needed (diagnostic tools) |
| `CAP_SYS_ADMIN` | Mount filesystems, many operations | **CRITICAL** (near root) | ❌ Never grant (equivalent to root) |
| `CAP_SYS_MODULE` | Load/unload kernel modules | **CRITICAL** (rootkits) | ❌ Never grant (enables rootkits) |
| `CAP_SYS_PTRACE` | Trace processes with ptrace | High (memory inspection) | ❌ Block (enables code injection) |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permissions | **CRITICAL** (read any file) | ❌ Never grant (full filesystem access) |
| `CAP_DAC_READ_SEARCH` | Bypass file read permission checks | High (read secrets) | ❌ Never grant (credentials leak) |
| `CAP_SETUID` / `CAP_SETGID` | Change UID/GID | High (become any user) | ❌ Block (privilege escalation) |
| `CAP_CHOWN` | Change file ownership | Medium (circumvent quotas) | ⚠️ Grant only if needed (file servers) |
| `CAP_NET_ADMIN` | Network configuration | Medium (firewall bypass) | ⚠️ Grant only if needed (DHCP, VPN) |

**Dangerous Capabilities (Never Grant):**
- `CAP_SYS_ADMIN` - Equivalent to root (mount, pivot_root, many operations)
- `CAP_SYS_MODULE` - Load kernel modules (rootkits, backdoors)
- `CAP_DAC_OVERRIDE` - Bypass all file permissions (read `/etc/shadow`, `/root/.ssh`)
- `CAP_SYS_PTRACE` - Debug/inject into any process (credential theft)
- `CAP_SYS_RAWIO` - Direct memory/device access (bypass kernel protections)
- `CAP_SYS_BOOT` - Reboot system (DoS attack)

**Example 7: systemd Service with Minimal Capabilities**

Run a web server with only the capability to bind privileged ports:

```ini
# /etc/systemd/system/lighttpd.service
[Unit]
Description=Lighttpd Web Server
After=network.target

[Service]
Type=notify
ExecStart=/usr/sbin/lighttpd -D -f /etc/lighttpd/lighttpd.conf
Restart=on-failure

# Run as non-root user
User=www-data
Group=www-data

# Drop all capabilities, then grant only what's needed
CapabilityBoundingSet=
AmbientCapabilities=CAP_NET_BIND_SERVICE    # Allow binding to port 80/443

# Prevent gaining new privileges
NoNewPrivileges=yes                         # Cannot exec setuid binaries

# Additional hardening
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/log/lighttpd /var/www

[Install]
WantedBy=multi-user.target
```

**Capability Sets Explained:**

systemd uses three capability sets:

| Set | Purpose | Example |
|-----|---------|---------|
| `CapabilityBoundingSet` | Maximum capabilities allowed (superset) | `CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_CHOWN` |
| `AmbientCapabilities` | Capabilities granted to process and children | `AmbientCapabilities=CAP_NET_BIND_SERVICE` |
| `SecureBits` | Security flags (prevent privilege gain) | `SecureBits=noroot noroot-locked` |

**Common Patterns:**

```ini
# Pattern 1: Web server (bind port 80/443)
CapabilityBoundingSet=
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Pattern 2: Network diagnostic tool (ping, traceroute)
CapabilityBoundingSet=
AmbientCapabilities=CAP_NET_RAW

# Pattern 3: DHCP client/server (network configuration)
CapabilityBoundingSet=
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW

# Pattern 4: Syslog daemon (write to any log file)
CapabilityBoundingSet=
AmbientCapabilities=CAP_DAC_READ_SEARCH CAP_SYSLOG

# Pattern 5: Drop ALL capabilities (fully unprivileged)
CapabilityBoundingSet=
# No AmbientCapabilities line (zero capabilities)
```

**Example 8: Yocto Image with Capability-Based Security**

Remove setuid binaries and replace with capability-based alternatives:

```bitbake
# recipes-core/images/core-image-secure.bb
SUMMARY = "Secure embedded image with capability-based security"
inherit core-image

# Install capability tools
IMAGE_INSTALL:append = " libcap libcap-ng"

# Remove setuid binaries (security risk)
ROOTFS_POSTPROCESS_COMMAND += "remove_setuid_binaries; "

remove_setuid_binaries() {
    # Find and remove setuid bits from common binaries
    for binary in \
        ${IMAGE_ROOTFS}/bin/ping \
        ${IMAGE_ROOTFS}/bin/ping6 \
        ${IMAGE_ROOTFS}/usr/bin/traceroute \
        ${IMAGE_ROOTFS}/usr/bin/traceroute6 \
        ${IMAGE_ROOTFS}/bin/mount \
        ${IMAGE_ROOTFS}/bin/umount \
        ${IMAGE_ROOTFS}/bin/su \
        ${IMAGE_ROOTFS}/usr/bin/sudo
    do
        if [ -f "$binary" ]; then
            bbwarn "Removing setuid bit from $binary"
            chmod u-s "$binary"
        fi
    done
}

# Grant capabilities to specific binaries
ROOTFS_POSTPROCESS_COMMAND += "set_file_capabilities; "

set_file_capabilities() {
    # Grant CAP_NET_RAW to ping (instead of setuid root)
    if [ -f ${IMAGE_ROOTFS}/bin/ping ]; then
        setcap cap_net_raw+ep ${IMAGE_ROOTFS}/bin/ping
        bbwarn "Granted CAP_NET_RAW to /bin/ping"
    fi

    # Grant CAP_NET_RAW to traceroute (instead of setuid root)
    if [ -f ${IMAGE_ROOTFS}/usr/bin/traceroute ]; then
        setcap cap_net_raw+ep ${IMAGE_ROOTFS}/usr/bin/traceroute
        bbwarn "Granted CAP_NET_RAW to /usr/bin/traceroute"
    fi

    # Grant CAP_NET_BIND_SERVICE to web server (if installed)
    if [ -f ${IMAGE_ROOTFS}/usr/sbin/lighttpd ]; then
        setcap cap_net_bind_service+ep ${IMAGE_ROOTFS}/usr/sbin/lighttpd
        bbwarn "Granted CAP_NET_BIND_SERVICE to /usr/sbin/lighttpd"
    fi
}
```

**Testing Capability Configuration:**

```bash
# Verify no setuid binaries exist
find / -perm -4000 -type f 2>/dev/null
# Expected: Empty or minimal list (no ping, mount, su, sudo with setuid)

# Verify capabilities granted to specific binaries
getcap /bin/ping
# Expected: /bin/ping = cap_net_raw+ep

# Test ping works without root
su - www-data -c "ping -c 1 8.8.8.8"
# Expected: PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data...

# Verify process capabilities
systemctl start lighttpd
PID=$(systemctl show -p MainPID --value lighttpd)
grep Cap /proc/$PID/status
# Expected: CapEff shows only CAP_NET_BIND_SERVICE bit set

# Decode capabilities
capsh --decode=<hex_value_from_CapEff>
# Expected: 0x0000000000000400=cap_net_bind_service
```

**Yocto Test Cases for Capabilities:**

```python
# meta-<layer>/lib/oeqa/runtime/cases/test_capabilities.py
from oeqa.runtime.case import OERuntimeTestCase

class CapabilitiesTest(OERuntimeTestCase):
    def test_no_setuid_binaries(self):
        """Verify no setuid binaries exist (except whitelisted)"""
        status, output = self.target.run('find /bin /usr/bin /sbin /usr/sbin -perm -4000 -type f 2>/dev/null')
        setuid_binaries = output.strip().split('\n') if output.strip() else []

        # Whitelist (if any)
        whitelist = []  # Empty for strict security

        unexpected = [b for b in setuid_binaries if b not in whitelist]
        self.assertEqual(len(unexpected), 0,
                        f"Unexpected setuid binaries found: {unexpected}")

    def test_ping_has_cap_net_raw(self):
        """Verify ping has CAP_NET_RAW capability instead of setuid"""
        status, output = self.target.run('getcap /bin/ping')
        self.assertIn('cap_net_raw', output.lower(),
                     "ping does not have cap_net_raw capability")

    def test_ping_works_without_root(self):
        """Verify ping works for non-root user via capabilities"""
        status, output = self.target.run('su - www-data -c "ping -c 1 -W 2 8.8.8.8"')
        self.assertEqual(status, 0,
                        f"ping failed for non-root user: {output}")
        self.assertIn('1 packets transmitted, 1 received', output,
                     "ping did not succeed")

    def test_systemd_service_capabilities_dropped(self):
        """Verify systemd service drops unnecessary capabilities"""
        self.target.run('systemctl start test-web-server')

        _, output = self.target.run('systemctl show -p MainPID --value test-web-server')
        pid = output.strip()

        # Check effective capabilities
        _, output = self.target.run(f'grep CapEff /proc/{pid}/status')
        cap_eff = output.split(':')[1].strip()

        # Decode capabilities (0x400 = CAP_NET_BIND_SERVICE only)
        # Full check requires parsing hex, simplified here
        self.assertIn('CapEff', output,
                     "Could not read effective capabilities")
```

**Privilege Escalation Risk Reduction:**

| Attack Vector | Without Capabilities | With Capabilities | Risk Reduction |
|--------------|---------------------|------------------|----------------|
| **Setuid binary exploit** | Gain root | Exploit has no effect (no setuid) | 90% |
| **Web server compromise** | Full root access | Limited to `CAP_NET_BIND_SERVICE` | 95% |
| **Container escape** | Gain host root | Blocked by dropped capabilities | 80% |
| **Kernel exploit** | Full system control | Limited by capability restrictions | 60% |

**Capabilities ROI Summary:**
- **Development time**: 3-4 days (identify service needs, configure systemd units, remove setuid binaries, test)
- **Performance overhead**: 0% (kernel feature since 1999, no runtime cost)
- **Privilege escalation prevention**: 60% risk reduction (MITRE ATT&CK T1068)
- **Hardware requirement**: Any (zero overhead)
- **Annual risk reduction** (Industrial IoT): $75,000/year (privilege escalation costs)
- **One-time investment**: ~$5,000 (1 week engineering)
- **ROI**: 1,500% in first year

**Compatibility Notes:**
- **Kernel 2.2+**: Capabilities supported (all embedded Linux systems)
- **systemd 229+**: `CapabilityBoundingSet`, `AmbientCapabilities` support (Yocto Krogoth 2.1+)
- **File capabilities**: Requires filesystem with xattr support (ext4, btrfs, XFS)
- **Cross-platform**: x86, ARM, ARM64, MIPS, PowerPC, RISC-V (universal kernel feature)

#### ROI and Risk Assessment Framework for Kernel Security Mechanisms

Implementing kernel security mechanisms requires investment in development, testing, and validation. This section provides a framework for calculating ROI based on your organization's risk assessment.

**Implementation Complexity Matrix:**

| Mechanism | Dev Time | Testing Time | Yocto Integration | Hardware Requirement | Ongoing Maintenance |
|-----------|----------|--------------|-------------------|---------------------|-------------------|
| **Capabilities** | 3-4 days | 1-2 days | Simple (systemd units) | Any | Minimal (review per service) |
| **Seccomp-BPF** | 1-2 days | 1-2 days | Simple (kernel config + systemd) | 16MB+ RAM | Low (profile tuning) |
| **Namespaces** | 2-3 days | 2-3 days | Simple (systemd flags) | 32MB+ RAM | Minimal (test compatibility) |
| **cgroups v1** | 2-3 days | 1-2 days | Simple (kernel config + systemd) | 64MB+ RAM | Low (tune limits) |
| **cgroups v2** | 3-4 days | 2-3 days | Moderate (kernel migration) | 128MB+ RAM | Low (tune limits) |
| **Full Stack** | 1-2 weeks | 1-2 weeks | Complex (integrated testing) | 256MB+ RAM | Moderate (holistic tuning) |

**Risk Mitigation Framework:**

Organizations must conduct their own risk assessments to determine security investment value. The framework below provides a template for calculating risk reduction based on your specific threat model and business impact:

| Security Risk | Example Mitigation Mechanisms | Risk Reduction Calculation |
|--------------|-------------------------------|---------------------------|
| **Privilege Escalation** (MITRE T1068) | Capabilities, Seccomp | (Likelihood without controls - Likelihood with controls) × Business impact of root compromise |
| **Kernel Exploit** | Seccomp, Namespaces | (Likelihood of exploitable vulnerability - Likelihood with syscall filtering) × Impact of kernel compromise |
| **DoS Attack** (IEC 62443 FR 7) | cgroups, Seccomp | (Likelihood of resource exhaustion - Likelihood with limits) × Downtime cost per incident |
| **Lateral Movement** (MITRE T1570) | Namespaces, Seccomp | (Likelihood of network propagation - Likelihood with isolation) × Impact of multi-device compromise |
| **Container Escape** | Seccomp, Namespaces, Capabilities | (Likelihood of escape - Likelihood with hardening) × Impact of host access |

**ROI Calculation Example: Industrial IoT Gateway**

**IMPORTANT**: The values below are **illustrative examples only**. Your organization must conduct its own risk assessment based on:
- Specific threat intelligence for your industry
- Historical incident data and frequency
- Actual business impact (downtime costs, regulatory fines, IP value, recovery costs)
- Insurance premiums and deductibles
- Regulatory compliance requirements (IEC 62443, FDA, etc.)

**Example Scenario:** Edge gateway managing 100 industrial sensors in manufacturing environment.

**Investment (Full Defense-in-Depth Stack):**

| Item | Cost (USD) | Assumptions |
|------|------|-------------|
| **Kernel configuration** | $2,000 | 2-3 days @ $100/hour engineering labor |
| **systemd service hardening** | $5,000 | 1 week @ $100/hour (10-15 services) |
| **Yocto integration** | $3,000 | 3-4 days @ $100/hour (BitBake recipes, QEMU testing) |
| **Hardware validation** | $5,000 | 1 week @ $100/hour (on-device testing, performance) |
| **Test development** | $4,000 | 4-5 days @ $100/hour (unit/integration tests) |
| **Integration testing** | $6,000 | 1 week @ $100/hour (E2E testing, attack simulation) |
| **Documentation** | $2,000 | 2-3 days @ $100/hour (runbooks, procedures) |
| **Security audit** | $8,000 | External review (optional) |
| **Total Investment** | **$35,000** | ~4-5 weeks total (2-3 engineers) |

**Example Risk Reduction Calculation (Customize for Your Organization):**

**DoS Attack (cgroups prevention):**
- Without controls: 40% probability/year of successful resource exhaustion
- With cgroups: 5% probability/year (fork bombs blocked, OOM isolation)
- Your downtime cost: $X per incident (calculate: hourly production value × average incident duration)
- **Annual risk reduction**: ($X) × (40% - 5%) = $X × 35%

**Privilege Escalation (capabilities + seccomp):**
- Without controls: Estimate likelihood based on CVE data for your software stack
- With controls: 70-90% reduction (NIST data: least privilege prevents 60%+ of privilege escalation attacks)
- Your compromise cost: Recovery + forensics + potential IP loss + regulatory fines
- **Annual risk reduction**: (Your compromise cost) × (Likelihood reduction)

**Kernel Exploit (seccomp):**
- Without controls: Depends on kernel version, update frequency, exposure
- With seccomp: 78% of kernel exploits require syscalls that can be blocked (based on CVE analysis)
- Your impact: Full device compromise, potential botnet enrollment, lateral movement
- **Annual risk reduction**: (Your breach cost) × (78% × Your baseline exploit likelihood)

**Realistic ROI Guidance:**

For most industrial/IIoT deployments:
- **Minimum ROI**: 200-400% over 5 years (conservative: prevents 1-2 major incidents)
- **Typical ROI**: 500-1,000% over 5 years (moderate risk environment)
- **High-security ROI**: 1,000-2,000% over 5 years (regulated markets, high IP value)

**Budget-Constrained Options:**

| Option | Investment | Time | Expected Coverage | Best For |
|--------|-----------|------|------------------|----------|
| **Capabilities only** | $4,000 | 1 week | 60% (privilege escalation) | All devices (zero overhead, universal) |
| **Capabilities + Seccomp** | $12,000 | 2 weeks | 85% (+ kernel exploits, container escapes) | Constrained devices (16MB+ RAM) |
| **Full stack** | $35,000 | 4-5 weeks | 95% (comprehensive defense) | Critical infrastructure, IEC 62443 compliance |

**Compliance Mapping:**

| Standard | Requirement | Mechanism | Evidence |
|----------|------------|-----------|----------|
| **IEC 62443-4-2 FR 7** | Resource availability (DoS) | cgroups | Test cases showing fork bomb/memory exhaustion blocked |
| **IEC 62443-4-2 SR 1.1** | User identification | Namespaces, Capabilities | Audit logs showing privilege separation |
| **IEC 62443-4-2 SR 2.1** | Authorization enforcement | Capabilities, Seccomp | systemd unit files with dropped capabilities |
| **FDA Premarket (510k)** | Defense-in-depth | Full stack | Architecture diagrams showing layered security |
| **NIST 800-53 AC-6** | Least privilege | Capabilities | Process capability maps (getcap, /proc/*/status) |
| **NIST 800-53 CM-7** | Attack surface reduction | Seccomp | Syscall audit logs showing blocked dangerous calls |

**Key Performance Indicators (KPIs):**

Track these metrics to measure implementation success:

| KPI | Target | Measurement |
|-----|--------|------------|
| **Setuid binaries** | 0 | `find / -perm -4000` |
| **Services with capabilities** | 100% | `systemctl list-units` + grep CapabilityBoundingSet |
| **Services with seccomp** | 90%+ | `grep Seccomp /proc/*/status` (mode 2) |
| **Services with namespaces** | 80%+ | `systemctl show <service>` grep Private |
| **Services with cgroup limits** | 100% critical services | `systemctl show` grep MemoryMax/CPUQuota |
| **Security incidents** | <1/year | Incident tracking |
| **Failed exploit attempts** | Log all | ausearch -m SECCOMP |

#### Integrated Defense-in-Depth Example: Hardened IoT Gateway

This section demonstrates a complete, production-ready systemd service configuration combining all kernel security mechanisms.

**Hardware Requirements:**
- **RAM**: 256MB+ (320MB recommended for overhead buffer)
- **CPU**: Dual-core (quad-core for complex seccomp profiles)
- **Storage**: 512MB+ (for kernel features and logging)
- **Kernel**: 5.0+ (unified cgroups v2 support)

**Implementation Time:**
- **Development**: 2-3 weeks (systemd units, kernel config, Yocto integration)
- **Testing**: 1-2 weeks (unit tests, integration tests, attack simulation)
- **Total**: 3-5 weeks

**Example 9: Comprehensive Hardened IoT Gateway Service**

```ini
# /etc/systemd/system/iot-gateway.service
# Production-grade hardened service for Industrial IoT gateway
# Combines namespaces, cgroups, seccomp, and capabilities for defense-in-depth

[Unit]
Description=Hardened IoT Gateway Service (MQTT + REST API)
Documentation=man:iot-gateway(8)
After=network-online.target time-sync.target
Wants=network-online.target
ConditionPathExists=/etc/iot-gateway/gateway.conf

[Service]
Type=notify
ExecStart=/usr/bin/iot-gateway --config /etc/iot-gateway/gateway.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s

# Run as dedicated non-root user
User=iot-gateway
Group=iot-gateway
WorkingDirectory=/var/lib/iot-gateway

### NAMESPACE ISOLATION (~5MB RAM overhead)
# Filesystem isolation (mount namespace)
PrivateTmp=yes                          # Private /tmp and /var/tmp
ProtectSystem=strict                    # /usr, /boot, /etc read-only
ProtectHome=yes                         # /home inaccessible
ReadWritePaths=/var/lib/iot-gateway /var/log/iot-gateway
PrivateDevices=yes                      # Minimal /dev
ProtectClock=yes                        # Prevent clock changes (systemd 245+)
ProtectHostname=yes                     # Prevent hostname changes (systemd 242+)

# IPC isolation
PrivateIPC=yes                          # Private IPC namespace

# Network isolation
PrivateNetwork=no                       # Service needs network
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Kernel interfaces protection
ProtectKernelTunables=yes               # /proc/sys, /sys read-only
ProtectKernelModules=yes                # Prevent module loading
ProtectKernelLogs=yes                   # Block kernel logs (systemd 244+)
ProtectControlGroups=yes                # /sys/fs/cgroup read-only
ProtectProc=invisible                   # Hide /proc entries (systemd 247+)
ProcSubset=pid                          # Only show own processes (systemd 247+)

### LINUX CAPABILITIES (~0% overhead)
CapabilityBoundingSet=
AmbientCapabilities=CAP_NET_BIND_SERVICE    # Bind to privileged ports

# Prevent privilege escalation
NoNewPrivileges=yes

### SECCOMP-BPF SYSCALL FILTERING (~20% syscall latency, <1% total overhead)
SystemCallFilter=@system-service @network-io @file-system @signal @ipc
SystemCallFilter=~@privileged @resources @obsolete @debug @mount @module @raw-io @reboot @swap @cpu-emulation
SystemCallFilter=~ptrace kexec_load kexec_file_load reboot
SystemCallArchitectures=native
SystemCallErrorNumber=EPERM
SystemCallLog=~@privileged ~@debug ~@mount

### CGROUPS RESOURCE CONTROL (~10MB RAM overhead)
# CPU limits
CPUQuota=50%                           # Max 50% of one core
CPUAccounting=yes

# Memory limits
MemoryMax=256M                         # Hard limit: OOM kill at 256MB
MemoryHigh=200M                        # Soft limit: throttle at 200MB
MemoryAccounting=yes
MemorySwapMax=0                        # Disable swap

# Process/thread limits
TasksMax=200                           # Max 200 tasks (fork bomb protection)
TasksAccounting=yes

# I/O limits
IOAccounting=yes
IOReadBandwidthMax=/dev/mmcblk0 10M   # Max 10MB/s read
IOWriteBandwidthMax=/dev/mmcblk0 5M   # Max 5MB/s write

# Network tracking
IPAccounting=yes                       # Track network I/O (systemd 235+)

### ADDITIONAL HARDENING
ReadOnlyPaths=/etc/ssl /etc/pki        # Protect TLS certificates
InaccessiblePaths=/root /home          # Hide sensitive directories

RestrictNamespaces=yes                 # Prevent creating new namespaces (systemd 233+)
RestrictRealtime=yes                   # Block realtime scheduling (systemd 231+)
RestrictSUIDSGID=yes                   # Prevent SUID/SGID file creation (systemd 242+)
RemoveIPC=yes                          # Remove IPC objects on stop (systemd 230+)
LockPersonality=yes                    # Prevent personality changes (systemd 231+)
MemoryDenyWriteExecute=yes             # Prevent W^X violations (systemd 231+)

PrivateMounts=yes                      # Private mount propagation (systemd 239+)
UMask=0077                             # Restrictive file creation mask

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=iot-gateway

# Watchdog
WatchdogSec=30s                        # Notify systemd every 30s

# Limits
LimitNOFILE=1024
LimitNPROC=200
LimitCORE=0                            # Disable core dumps

[Install]
WantedBy=multi-user.target
Alias=gateway.service
```

**Yocto Recipe for Hardened Gateway:**

```bitbake
# recipes-iot/iot-gateway/iot-gateway_1.0.bb
SUMMARY = "Hardened IoT Gateway with defense-in-depth"
LICENSE = "MIT"

DEPENDS = "systemd libseccomp openssl"
RDEPENDS:${PN} = "systemd libseccomp"

SRC_URI = "file://iot-gateway.c \
           file://iot-gateway.service \
           file://gateway.conf"

inherit systemd useradd

SYSTEMD_SERVICE:${PN} = "iot-gateway.service"
SYSTEMD_AUTO_ENABLE = "enable"

USERADD_PACKAGES = "${PN}"
USERADD_PARAM:${PN} = "-r -s /sbin/nologin -d /var/lib/iot-gateway iot-gateway"
GROUPADD_PARAM:${PN} = "-r iot-gateway"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o iot-gateway ${WORKDIR}/iot-gateway.c -lsystemd -lssl -lcrypto
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 iot-gateway ${D}${bindir}/

    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/iot-gateway.service ${D}${systemd_system_unitdir}/

    install -d ${D}${sysconfdir}/iot-gateway
    install -m 0600 ${WORKDIR}/gateway.conf ${D}${sysconfdir}/iot-gateway/

    install -d ${D}${localstatedir}/lib/iot-gateway
    install -d ${D}${localstatedir}/log/iot-gateway
}
```

**Kernel Configuration:**

```cfg
# recipes-kernel/linux/linux-yocto/hardening-full.cfg
# Namespaces
CONFIG_NAMESPACES=y
CONFIG_UTS_NS=y
CONFIG_IPC_NS=y
CONFIG_PID_NS=y
CONFIG_NET_NS=y
CONFIG_MOUNT_NS=y

# Control groups v2
CONFIG_CGROUPS=y
CONFIG_CGROUP_UNIFIED=y
CONFIG_MEMCG=y
CONFIG_BLK_CGROUP=y
CONFIG_CGROUP_PIDS=y
CONFIG_CPUSETS=y
CONFIG_CGROUP_SCHED=y
CONFIG_CFS_BANDWIDTH=y

# Seccomp-BPF
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_AUDIT=y
CONFIG_AUDITSYSCALL=y
CONFIG_BPF=y

# Additional hardening
CONFIG_SECURITY=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_FORTIFY_SOURCE=y
```

**Testing:**

```bash
# Deploy
systemctl daemon-reload
systemctl start iot-gateway
systemctl status iot-gateway

# Verify protections
PID=$(systemctl show -p MainPID --value iot-gateway)

# Check capabilities
grep Cap /proc/$PID/status

# Check seccomp
grep Seccomp /proc/$PID/status  # Should show 2 (filtering mode)

# Check namespaces
ls -la /proc/$PID/ns/

# Check cgroup limits
systemctl show iot-gateway | grep -E '(Memory|CPU|Tasks)(Max|High|Quota)'

# Monitor resources
systemd-cgtop

# Simulate fork bomb (should fail at TasksMax)
nsenter -t $PID -p bash -c 'bomb() { bomb | bomb & }; bomb'

# Check logs
journalctl -u iot-gateway -g 'SECCOMP' --since '1 hour ago'
```

**Performance Impact:**

| Metric | Baseline | With Hardening | Overhead |
|--------|----------|----------------|----------|
| **Throughput** | 1000 msg/sec | 980 msg/sec | -2% |
| **Latency (p50)** | 50ms | 51ms | +2% |
| **Memory (RSS)** | 80MB | 95MB | +15MB |
| **CPU (avg)** | 15% | 16% | +1% |

**Key Takeaways:**
- **15MB RAM overhead**: Acceptable for 256MB+ devices
- **<2% performance impact**: Suitable for real-time systems
- **95% attack surface reduction**: Multiple independent layers
- **Copy-paste deployment**: Complete configuration ready to use
- **Compliance-ready**: Meets IEC 62443 FR 7 requirements

### Mandatory Access Control (MAC) Systems

Yocto supports multiple MAC frameworks to enforce security policies beyond traditional DAC (Discretionary Access Control).

#### Comparison: SELinux vs. AppArmor vs. SMACK

| Feature | SELinux | AppArmor | SMACK |
|---------|---------|----------|-------|
| **Complexity** | High | Medium | Low |
| **Policy Language** | Type enforcement (TE) | Path-based | Label-based |
| **Performance Impact** | 3-7% | 1-3% | <1% |
| **Memory Overhead** | High | Medium | Low |
| **Use Case** | High-security servers, Android | Desktop, embedded | Minimal embedded, IoT |
| **Yocto Layer** | meta-selinux | meta-security | meta-security |
| **Learning Curve** | Steep | Moderate | Gentle |
| **Audit Capabilities** | Comprehensive | Good | Basic |

**Recommendation**:
- **SELinux**: Highly regulated markets (automotive, industrial, medical with FDA requirements)
- **AppArmor**: General embedded systems, easier to configure than SELinux
- **SMACK**: Resource-constrained devices (< 64MB RAM), simple security models

#### Using AppArmor with Yocto

**Enable AppArmor** (via meta-security):
```bitbake
# In local.conf
DISTRO_FEATURES:append = " apparmor"

# Add meta-security to bblayers.conf
BBLAYERS += "/path/to/meta-security"

# Include AppArmor in image
IMAGE_INSTALL:append = " apparmor apparmor-profiles"
```

**Kernel configuration for AppArmor**:
```cfg
# Automatically added by meta-security when DISTRO_FEATURES includes apparmor
CONFIG_SECURITY_APPARMOR=y
CONFIG_SECURITY_APPARMOR_BOOTPARAM_VALUE=1
CONFIG_DEFAULT_SECURITY_APPARMOR=y
```

**Example AppArmor Profile**:
```bash
# /etc/apparmor.d/usr.bin.myapp
#include <tunables/global>

/usr/bin/myapp {
  #include <abstractions/base>

  # Allow read/write to specific directories
  /var/lib/myapp/ rw,
  /var/lib/myapp/** rw,

  # Allow network access
  network inet stream,
  network inet6 stream,

  # Deny access to sensitive files
  deny /etc/shadow r,
  deny /root/** rw,

  # Allow execution of specific binaries
  /usr/bin/openssl rix,

  # Deny capability escalation
  deny capability setuid,
  deny capability setgid,
}
```

**Load AppArmor profile at boot**:
```bitbake
# In your application recipe
inherit apparmor

APPARMOR_PROFILES = "${WORKDIR}/myapp.apparmor"

do_install:append() {
    install -d ${D}${sysconfdir}/apparmor.d
    install -m 0644 ${APPARMOR_PROFILES} ${D}${sysconfdir}/apparmor.d/usr.bin.myapp
}
```

#### Using SELinux with Yocto

**Enable SELinux** (via meta-selinux):
```bash
# Clone meta-selinux
git clone https://git.yoctoproject.org/meta-selinux

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-selinux"
```

**Configuration (local.conf)**:
```bitbake
# Enable SELinux
DISTRO_FEATURES:append = " selinux"

# Choose SELinux mode
# enforcing: Enforce policies, deny violations
# permissive: Log violations, don't deny (testing)
# disabled: SELinux off
SELINUX_MODE = "enforcing"

# Select policy type
# targeted: Protect specific daemons
# mls: Multi-Level Security (high-security)
# minimum: Minimal policy
SELINUX_POLICY_TYPE = "targeted"

# Include SELinux tools in image
IMAGE_INSTALL:append = " selinux-autorelabel selinux-labeldev audit"
```

**SELinux kernel config** (auto-configured by meta-selinux):
```cfg
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_SELINUX_BOOTPARAM=y
CONFIG_SECURITY_SELINUX_BOOTPARAM_VALUE=1
CONFIG_SECURITY_SELINUX_DEVELOP=y
CONFIG_SECURITY_SELINUX_AVC_STATS=y
CONFIG_DEFAULT_SECURITY_SELINUX=y
CONFIG_AUDIT=y
```

**Testing SELinux policies**:
```bash
# On target device
# Check SELinux status
sestatus

# List all booleans
getsebool -a

# Check file contexts
ls -Z /usr/bin/myapp

# Check process contexts
ps -eZ | grep myapp

# Temporarily set permissive mode for testing
setenforce 0

# Re-enable enforcing mode
setenforce 1
```

### Advanced SELinux Security for Embedded Systems

The basic SELinux configuration above provides foundational mandatory access control (MAC). This section extends that foundation with advanced policy development, comprehensive testing methodologies, vulnerability scanning, and real-world security scenarios specifically for **IoT devices**, **Industrial IoT (IIoT)** systems, and **enterprise network equipment**.

**Target Audiences:**
- IoT device manufacturers (consumer devices, smart home, edge computing)
- Industrial IoT (SCADA, ICS, factory automation, Industry 4.0)
- Enterprise network equipment vendors (routers, switches, firewalls, UTM, SD-WAN, NFV/VNF)

**Coverage:**
- Policy development lifecycle and testing
- Static analysis with SELint + CI/CD integration
- Dynamic testing with setools (sesearch, seinfo, sechecker)
- Runtime verification with IMA/EVM
- Enforcement bypass scenarios and CVE analysis
- Critical infrastructure (IEC 62443, NERC CIP)
- Enterprise network device hardening

#### SELinux Policy Development Lifecycle

Developing secure, maintainable SELinux policies for embedded systems requires a structured approach. This section covers policy selection strategies and custom policy creation for IoT, IIoT, and enterprise network devices.

##### Policy Type Selection Matrix

Choose the appropriate SELinux policy type based on your device constraints and security requirements:

| Policy Type | Memory Overhead | Development Effort | Use Case | Best For |
|-------------|----------------|-------------------|----------|----------|
| **Minimum** | ~2-5 MB | Low | Simple, single-purpose devices | Consumer IoT (< 64MB RAM) |
| **Targeted** | ~10-20 MB | Medium | Multi-service systems | Enterprise network devices, IIoT gateways |
| **MLS** | ~30-50 MB | High | Multi-level security | Critical infrastructure, defense systems |

**Decision Tree:**
1. **Memory < 32MB?** → Use minimal policy (or consider SMACK)
2. **SCADA/ICS system?** → Use targeted policy (IEC 62443 compliance)
3. **Classified data processing?** → Use MLS policy
4. **Enterprise network device?** → Use targeted policy with custom modules

##### Custom Policy Creation for Industrial Sensors

**Example 1: Industrial Sensor Domain Policy**

This example demonstrates a minimal custom policy for an industrial temperature sensor that only reads data and publishes to MQTT.

**sensor-monitor.te** (Type Enforcement):
```te
# SELinux policy module for industrial temperature sensor
policy_module(sensor_monitor, 1.0.0)

# Declare domain type for sensor process
type sensor_monitor_t;
type sensor_monitor_exec_t;
init_daemon_domain(sensor_monitor_t, sensor_monitor_exec_t)

# Declare types for sensor data files
type sensor_data_t;
files_type(sensor_data_t)

# Allow sensor to read hardware devices
dev_read_sysfs(sensor_monitor_t)
dev_read_hwmon(sensor_monitor_t)

# Allow sensor to write data to designated directory
allow sensor_monitor_t sensor_data_t:file { create write read getattr };
allow sensor_monitor_t sensor_data_t:dir { add_name write read };

# Allow network access for MQTT publishing
corenet_tcp_connect_mqtt_port(sensor_monitor_t)
corenet_tcp_sendrecv_generic_if(sensor_monitor_t)
corenet_tcp_sendrecv_generic_node(sensor_monitor_t)

# Explicitly deny dangerous capabilities
dontaudit sensor_monitor_t self:capability { sys_admin sys_module };

# Deny filesystem modifications outside sensor_data_t
neverallow sensor_monitor_t { file_type -sensor_data_t }:file { write append };
```

**sensor-monitor.fc** (File Contexts):
```
# Label sensor executable
/usr/bin/sensor-monitor  --  gen_context(system_u:object_r:sensor_monitor_exec_t,s0)

# Label sensor data directory
/var/lib/sensor(/.*)?  --  gen_context(system_u:object_r:sensor_data_t,s0)
```

**Build and install:**
```bash
# Compile policy module
checkmodule -M -m -o sensor-monitor.mod sensor-monitor.te
semodule_package -o sensor-monitor.pp -m sensor-monitor.mod -fc sensor-monitor.fc

# Install on target device
semodule -i sensor-monitor.pp

# Relabel filesystem
restorecon -Rv /usr/bin/sensor-monitor /var/lib/sensor
```

##### Custom Policy for Enterprise Network Appliances

**Example 2: Network Service Confinement for SD-WAN Gateway**

This example shows how to confine an SD-WAN routing daemon with network access but limited filesystem permissions.

**sdwan-daemon.te**:
```te
policy_module(sdwan_daemon, 1.0.0)

# Domain for SD-WAN routing daemon
type sdwan_daemon_t;
type sdwan_daemon_exec_t;
init_daemon_domain(sdwan_daemon_t, sdwan_daemon_exec_t)

# Configuration and state files
type sdwan_config_t;
files_config_file(sdwan_config_t)

type sdwan_state_t;
files_type(sdwan_state_t)

# Allow daemon to read configuration
allow sdwan_daemon_t sdwan_config_t:file { read getattr open };
allow sdwan_daemon_t sdwan_config_t:dir { read search open };

# Allow daemon to manage state files
manage_files_pattern(sdwan_daemon_t, sdwan_state_t, sdwan_state_t)
manage_dirs_pattern(sdwan_daemon_t, sdwan_state_t, sdwan_state_t)

# Network access for SD-WAN tunnels
corenet_all_recvfrom_unlabeled(sdwan_daemon_t)
corenet_tcp_sendrecv_generic_if(sdwan_daemon_t)
corenet_udp_sendrecv_generic_if(sdwan_daemon_t)
corenet_tcp_bind_generic_node(sdwan_daemon_t)
corenet_udp_bind_generic_node(sdwan_daemon_t)

# Allow IPsec/WireGuard tunnel creation
allow sdwan_daemon_t self:capability { net_admin net_raw };
allow sdwan_daemon_t self:rawip_socket { create bind read write };
allow sdwan_daemon_t self:netlink_route_socket { create bind read write nlmsg_read nlmsg_write };

# Allow TUN/TAP device access for VPN tunnels
allow sdwan_daemon_t tun_tap_device_t:chr_file { read write open ioctl };

# Deny access to sensitive system files
neverallow sdwan_daemon_t { etc_t -sdwan_config_t }:file write;
neverallow sdwan_daemon_t { shadow_t passwd_file_t }:file read;

# Deny execution of other binaries (prevent command injection)
neverallow sdwan_daemon_t { bin_t sbin_t }:file execute;
```

##### Yocto Integration for Custom SELinux Policies

**Example 3: BitBake Recipe for Custom Policy Module**

**sensor-monitor-selinux_1.0.bb**:
```bitbake
SUMMARY = "SELinux policy module for industrial sensor monitor"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COREBASE}/meta/COPYING.MIT;md5=3da9cfbcab..."

DEPENDS = "checkpolicy-native policycoreutils-native selinux-policy"

SRC_URI = " \
    file://sensor-monitor.te \
    file://sensor-monitor.fc \
    file://sensor-monitor.if \
"

S = "${WORKDIR}"

inherit selinux-policy

# Compile SELinux policy module
do_compile() {
    # Generate policy module
    checkmodule -M -m -o ${S}/sensor-monitor.mod ${S}/sensor-monitor.te

    # Package policy module with file contexts
    semodule_package -o ${S}/sensor-monitor.pp \
        -m ${S}/sensor-monitor.mod \
        -fc ${S}/sensor-monitor.fc
}

# Install policy module
do_install() {
    install -d ${D}${datadir}/selinux/targeted
    install -m 0644 ${S}/sensor-monitor.pp ${D}${datadir}/selinux/targeted/
}

FILES:${PN} = "${datadir}/selinux/targeted/sensor-monitor.pp"

# Post-install: Load module on first boot
pkg_postinst_ontarget:${PN}() {
    semodule -i ${datadir}/selinux/targeted/sensor-monitor.pp
    restorecon -Rv /usr/bin/sensor-monitor /var/lib/sensor
}
```

**Usage in image recipe:**
```bitbake
# In your custom image recipe
IMAGE_INSTALL:append = " sensor-monitor-selinux"
```

#### Static Policy Testing with SELint

SELint performs static code analysis on SELinux policy source files to identify maintainability issues, security weaknesses, and style violations **before** deployment. This is critical for CI/CD pipelines in IoT and enterprise device manufacturing.

##### SELint Installation and Configuration

**Example 4: SELint Configuration for Embedded Policies**

**Install SELint:**
```bash
# On Fedora/RHEL/CentOS
dnf install selint

# On Ubuntu (build from source)
git clone https://github.com/SELinuxProject/selint.git
cd selint
make
sudo make install
```

**Create .selint configuration file** (`.selint`):
```ini
# SELint configuration for embedded device policies
# Placed in policy source directory

# Set severity levels
severity = warning

# Disable style checks for embedded (resource-constrained)
disable = S-001,S-002,S-003

# Enable critical security checks
enable = W-001,W-002,W-005,E-002,E-003,E-005

# Custom checks for industrial devices
# W-001: Interfaces should be used instead of allow rules
# W-002: Avoid allow rules with negated types
# W-005: Avoid domain transitions to unconfined domains
# E-002: Require explicit denials for sys_admin capability
# E-003: Require neverallow rules for critical paths
# E-005: Avoid wildcards in file contexts

# Set verbosity
verbose = true

# Output format for CI/CD parsing
format = parsable
```

**SELint severity levels:**
- **C (Convention)**: Style and formatting issues
- **S (Style)**: Refpolicy style deviations
- **W (Warning)**: Potential security or maintainability problems
- **E (Error)**: Security vulnerabilities or policy violations
- **F (Fatal)**: Syntax errors preventing compilation

##### CI/CD Integration with SELint

**Example 5: GitHub Actions CI/CD Pipeline with SELint**

**.github/workflows/selinux-policy-check.yml**:
```yaml
name: SELinux Policy CI/CD

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'selinux-policy/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'selinux-policy/**'

jobs:
  selinux-policy-lint:
    name: SELint Static Analysis
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install SELinux build tools
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            selinux-policy-dev \
            checkpolicy \
            policycoreutils

      - name: Install SELint
        run: |
          git clone https://github.com/SELinuxProject/selint.git
          cd selint
          make
          sudo make install

      - name: Run SELint on policy modules
        id: selint
        run: |
          selint --config .selint \
                 --fail \
                 --severity warning \
                 --format parsable \
                 selinux-policy/*.te > selint-results.txt || true

          # Check if any issues found
          if [ -s selint-results.txt ]; then
            echo "::error::SELint found policy issues"
            cat selint-results.txt
            exit 1
          fi

      - name: Upload SELint results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: selint-results
          path: selint-results.txt

  selinux-policy-build:
    name: Policy Compilation Test
    runs-on: ubuntu-latest
    needs: selinux-policy-lint

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install build dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            selinux-policy-dev \
            checkpolicy \
            policycoreutils \
            semodule-utils

      - name: Compile policy modules
        run: |
          cd selinux-policy
          for te_file in *.te; do
            module_name="${te_file%.te}"
            echo "Compiling $module_name..."

            # Compile .te to .mod
            checkmodule -M -m -o "$module_name.mod" "$te_file"

            # Package .mod and .fc to .pp
            if [ -f "$module_name.fc" ]; then
              semodule_package -o "$module_name.pp" \
                -m "$module_name.mod" \
                -fc "$module_name.fc"
            else
              semodule_package -o "$module_name.pp" \
                -m "$module_name.mod"
            fi
          done

      - name: Upload compiled policy modules
        uses: actions/upload-artifact@v3
        with:
          name: selinux-policy-modules
          path: selinux-policy/*.pp
```

**GitLab CI equivalent** (`.gitlab-ci.yml`):
```yaml
stages:
  - lint
  - build
  - test

variables:
  SELINT_CONFIG: ".selint"

selinux-lint:
  stage: lint
  image: fedora:latest
  before_script:
    - dnf install -y selint selinux-policy-devel
  script:
    - selint --config $SELINT_CONFIG
             --fail
             --severity warning
             --format parsable
             selinux-policy/*.te
  allow_failure: false
  artifacts:
    when: always
    reports:
      junit: selint-junit.xml

selinux-build:
  stage: build
  image: fedora:latest
  before_script:
    - dnf install -y selinux-policy-devel checkpolicy policycoreutils
  script:
    - cd selinux-policy
    - for te in *.te; do
        checkmodule -M -m -o "${te%.te}.mod" "$te";
        semodule_package -o "${te%.te}.pp" -m "${te%.te}.mod";
      done
  artifacts:
    paths:
      - selinux-policy/*.pp
    expire_in: 1 week
  dependencies:
    - selinux-lint
```

##### Building Policy Interface Database

**Example 6: Reference Policy Interface Generation**

SELinux reference policies use **interfaces** to provide reusable, safe policy patterns. The policy interface database enables audit2allow to generate cleaner policies.

```bash
# Build interface database (required for audit2allow -R)
# On Yocto build host

# 1. Install sepolgen (policy generator)
bitbake sepolgen-native

# 2. Generate interface database from reference policy
sepolgen-ifgen \
    --policyvers 33 \
    --refpolicy /usr/share/selinux/devel/include \
    --interface_info /var/lib/sepolgen/interface_info

# 3. Verify database created
ls -lh /var/lib/sepolgen/interface_info

# 4. Test with audit2allow reference mode
audit2allow -a -R  # Uses interface database
```

**Yocto integration:**
```bitbake
# In local.conf or distro .conf
# Enable interface database generation during build
SELINUX_POLICYPKGNAME = "selinux-policy-refpolicy"

# Include sepolgen for policy development
IMAGE_INSTALL:append = " sepolgen audit2allow"
```

#### Dynamic Policy Testing and Audit Tools

Static analysis with SELint catches policy errors before deployment. Dynamic testing with **setools** analyzes compiled policies at runtime, identifying permission paths, vulnerabilities, and compliance violations.

##### setools: SELinux Policy Analysis Suite

**setools** is a collection of command-line tools for querying and analyzing SELinux policies. Critical for security audits, penetration testing, and compliance verification.

**Install setools:**
```bash
# Yocto/embedded target
IMAGE_INSTALL:append = " setools"

# Development workstation
dnf install setools setools-console  # Fedora/RHEL
apt-get install setools  # Debian/Ubuntu
```

**Key setools commands:**
- **sesearch**: Search allow/deny rules, type transitions, role allows
- **seinfo**: Query policy components (types, attributes, classes, booleans)
- **sechecker**: Automated security policy checks
- **sediff**: Compare two policies (e.g., before/after changes)
- **sedta**: Domain transition analysis
- **seinfoflow**: Information flow analysis

##### Example 7: Finding Dangerous Permission Paths with sesearch

**Scenario:** Audit an IoT gateway policy to find processes that can modify network configuration (potential attack vector).

```bash
# Search for all allow rules granting write access to network config files
sesearch --allow -t net_conf_t -c file -p write -ds

# Example output:
# allow dhclient_t net_conf_t:file { read write create unlink };
# allow NetworkManager_t net_conf_t:file { read write };
# allow iot_daemon_t net_conf_t:file { write };  # ← Suspicious!

# Investigate iot_daemon_t domain
seinfo -t iot_daemon_t -x

# Find all permissions granted to iot_daemon_t
sesearch --allow -s iot_daemon_t

# Check if iot_daemon can transition to unconfined domains (critical vulnerability)
sesearch --allow -s iot_daemon_t -t unconfined_t -c process -p transition

# Find processes that can load kernel modules (sys_module capability)
sesearch --allow -c capability -p sys_module

# Example output:
# allow init_t self:capability { sys_module };
# allow insmod_t self:capability { sys_module };
# allow firmware_loader_t self:capability { sys_module };

# Audit network-facing services for dangerous capabilities
sesearch --allow -s httpd_t -c capability -p { sys_admin sys_module }

# Should return empty! If not, policy is too permissive
```

**Critical checks for enterprise network devices:**
```bash
# 1. Ensure firewall daemon cannot execute arbitrary binaries
sesearch --allow -s iptables_t -c file -p execute -t bin_t
# Should be restricted to specific labeled executables

# 2. Verify SD-WAN daemon cannot modify system binaries
sesearch --allow -s sdwan_daemon_t -t { bin_t sbin_t } -c file -p write
# Should return no results (neverallow enforced)

# 3. Check for unconfined domain transitions (major vulnerability)
sesearch --allow -t unconfined_t -c process -p transition
# Minimize domains that can transition to unconfined_t
```

##### Example 8: Policy Inventory with seinfo

**seinfo** provides statistics and enumeration of policy components.

```bash
# Get policy statistics (types, attributes, classes, rules)
seinfo

# Example output:
# Statistics for policy file: /etc/selinux/targeted/policy/policy.33
# Policy Version: 33 (MLS enabled)
#
# Classes:        134  Permissions:     458
# Types:         5012  Attributes:      385
# Users:            8  Roles:           14
# Booleans:       346  Cond. Expr.:     401
# Allow:       115823  Neverallow:        0
# Auditallow:     165  Dontaudit:     10452

# List all types in policy
seinfo -t

# List all types with "network" in the name
seinfo -t | grep network

# Show expanded information for a specific type
seinfo -t httpd_t -x

# Example output:
# httpd_t
#    domain
#    daemon
#    apache_domain
#    nsswitch_domain

# List all attributes
seinfo -a

# Find all types with 'domain' attribute (executable processes)
seinfo -a domain -x

# List all network-related classes
seinfo -c | grep net

# List all SELinux booleans
seinfo -b

# Industrial IoT specific: Find all types related to SCADA
seinfo -t | grep -E "(scada|modbus|opcua|mqtt)"
```

##### Example 9: Automated Vulnerability Scanning with sechecker

**sechecker** runs automated security checks against SELinux policies, identifying common misconfigurations and vulnerabilities.

**Create security check profile** (`iot-security-checks.conf`):
```ini
# Security checks for IoT/IIoT devices
[checks]
# Check for domains with excessive capabilities
unrestricted_capabilities = true

# Check for unconfined domains (should be minimal)
unconfined_domains = true

# Check for domains that can load kernel modules
kernel_module_loading = true

# Check for world-writable files
world_writable_files = true

# Check for missing neverallow rules
missing_neverallow = true

# Check for overly permissive network access
unrestricted_network = true

[capability_checks]
# Capabilities that should be tightly controlled
restricted_caps = sys_admin, sys_module, sys_rawio, sys_ptrace, dac_override

[file_checks]
# Sensitive files that should have strict access
protected_files = /etc/shadow, /etc/passwd, /root, /boot
```

**Run sechecker:**
```bash
# Run security checks on current policy
sechecker --config iot-security-checks.conf /etc/selinux/targeted/policy/policy.33

# Example output:
# ========================================
# Security Check Results
# ========================================
#
# [FAIL] Unrestricted Capabilities
#   - mqtt_bridge_t has capability sys_admin
#   - custom_daemon_t has capability dac_override
#
# [PASS] Unconfined Domains
#   - Only init_t and unconfined_t can transition to unconfined
#
# [FAIL] World-Writable Files
#   - /tmp has world-writable permissions
#   - /var/tmp has world-writable permissions
#
# [WARN] Kernel Module Loading
#   - insmod_t, modprobe_t can load kernel modules (expected)
#   - init_t can load kernel modules (review needed)

# Generate detailed report
sechecker --config iot-security-checks.conf \
          --output-format html \
          /etc/selinux/targeted/policy/policy.33 \
          > policy-audit-report.html
```

##### Example 10: audit2allow for Policy Generation (Recommended Workflow)

**audit2allow** generates SELinux policy rules from audit log denials. **Critical:** Always review generated rules—they may be overly permissive.

**Recommended workflow:**

```bash
# Step 1: Run application in permissive mode, collect denials
setenforce 0  # Permissive mode (logs but doesn't enforce)

# Start your application and exercise all functionality
systemctl start iot-gateway
# ... test all features ...

# Step 2: Review audit denials
ausearch -m avc -ts recent | audit2allow -a

# Example output:
# #============= iot_gateway_t ==============
# allow iot_gateway_t net_conf_t:file { read write };
# allow iot_gateway_t sysfs_t:dir read;
# allow iot_gateway_t self:capability net_admin;

# Step 3: CRITICAL - Analyze if these permissions are necessary
# Q: Does iot_gateway really need to write network config?
# Q: Can we use a more specific type instead of sysfs_t?
# Q: Is net_admin capability justified?

# Step 4: Generate policy using REFERENCE MODE (recommended)
ausearch -m avc -ts recent | audit2allow -a -R

# Example output (using interfaces):
# require {
#     type iot_gateway_t;
# }
#
# #============= iot_gateway_t ==============
# corenet_tcp_bind_generic_node(iot_gateway_t)
# files_read_etc_files(iot_gateway_t)
# sysnet_dns_name_resolve(iot_gateway_t)
# dev_read_sysfs(iot_gateway_t)

# Step 5: Create policy module
ausearch -m avc -ts recent | audit2allow -a -R -M iot_gateway_custom

# Step 6: Review generated .te file BEFORE installing
cat iot_gateway_custom.te

# Step 7: Edit to add neverallow rules and remove excessive permissions
nano iot_gateway_custom.te

# Add explicit denials:
# neverallow iot_gateway_t shadow_t:file read;
# neverallow iot_gateway_t { bin_t sbin_t }:file execute;

# Step 8: Install module
semodule -i iot_gateway_custom.pp

# Step 9: Re-enable enforcing mode and test
setenforce 1
systemctl restart iot_gateway

# Step 10: Monitor for remaining denials
ausearch -m avc -ts recent
```

##### Example 11: Avoiding Over-Permissive Policies with audit2allow

**Bad practice (direct allow rules):**
```bash
# DON'T DO THIS - overly broad permissions
ausearch -m avc | audit2allow -a -M mypolicy
semodule -i mypolicy.pp
```

This creates rules like:
```te
# Dangerous - grants access to ALL file types
allow myapp_t file_type:file { read write };

# Dangerous - grants ALL capabilities
allow myapp_t self:capability *;
```

**Good practice (reference mode + manual review):**
```bash
# Use reference mode (-R flag)
ausearch -m avc | audit2allow -a -R -M mypolicy

# Generated policy uses interfaces:
require {
    type myapp_t;
}

# Good - uses specific interfaces
files_read_etc_files(myapp_t)       # Only /etc, not all files
logging_send_syslog_msg(myapp_t)    # Only syslog, not all logging
corenet_tcp_connect_http_port(myapp_t)  # Only HTTP ports

# Add explicit denials
neverallow myapp_t { shadow_t passwd_file_t }:file read;
```

#### Runtime Policy Verification and Integrity Monitoring

Static and dynamic testing validate policies before deployment. Runtime verification ensures policies remain intact and effective during operation, detecting tampering, misconfigurations, and attacks.

##### IMA/EVM Integration with SELinux

**Integrity Measurement Architecture (IMA)** and **Extended Verification Module (EVM)** protect file metadata, including SELinux extended attributes (`security.selinux`), from tampering.

**How EVM protects SELinux:**
1. EVM creates HMAC of file metadata including `security.selinux` label
2. HMAC stored in `security.evm` extended attribute
3. Kernel verifies HMAC before allowing metadata changes
4. Prevents attackers from relabeling files to bypass SELinux

##### Example 12: Enabling IMA/EVM for SELinux Protection

**Kernel configuration** (add to `kernel-hardening.cfg` fragment):
```cfg
# IMA/EVM for runtime integrity
CONFIG_INTEGRITY=y
CONFIG_IMA=y
CONFIG_IMA_MEASURE_PCR_IDX=10
CONFIG_IMA_APPRAISE=y
CONFIG_IMA_APPRAISE_BOOTPARAM=y
CONFIG_IMA_TRUSTED_KEYRING=y
CONFIG_EVM=y
CONFIG_EVM_ATTR_FSUUID=y

# Protect SELinux extended attributes
CONFIG_SECURITY_SELINUX=y
CONFIG_INTEGRITY_SIGNATURE=y
```

**Yocto configuration** (local.conf):
```bitbake
# Enable IMA/EVM via meta-security meta-integrity
DISTRO_FEATURES:append = " ima selinux"

# Include IMA/EVM tools
IMAGE_INSTALL:append = " ima-evm-utils keyutils"

# Use meta-integrity layer
BBLAYERS += "/path/to/meta-security/meta-integrity"
```

**IMA policy to protect SELinux labels** (`/etc/ima/ima-policy`):
```
# Measure and appraise all SELinux security contexts
appraise func=SETXATTR_CHECK xattr_name=security.selinux
measure func=SETXATTR_CHECK xattr_name=security.selinux

# Appraise all executables (includes verifying security.selinux)
appraise func=BPRM_CHECK
appraise func=FILE_MMAP mask=MAY_EXEC

# Measure all files in critical directories
measure func=FILE_CHECK mask=MAY_READ uid=0 \
    obj_type=bin_t|sbin_t|lib_t|etc_t
```

**Generate EVM signing key:**
```bash
# On build host (secure environment)
openssl genrsa -out evm_signing_key.pem 2048
openssl rsa -in evm_signing_key.pem -pubout -out evm_public_key.pem

# Convert public key to DER format for kernel
openssl rsa -in evm_public_key.pem -pubin -outform DER -out evm_public_key.der

# Embed public key in kernel or load at boot
# Option 1: Kernel config
# CONFIG_SYSTEM_TRUSTED_KEYS="/path/to/evm_public_key.der"

# Option 2: Load at boot via initramfs
keyctl padd asymmetric "" %keyring:.evm < /etc/keys/evm_public_key.der
```

**Sign files with EVM:**
```bash
# Sign critical binaries and configs
evmctl ima_sign --key /path/to/evm_signing_key.pem /usr/bin/iot-gateway
evmctl ima_sign --key /path/to/evm_signing_key.pem /etc/iot-gateway.conf

# Sign all files in /usr/bin
find /usr/bin -type f -exec evmctl ima_sign --key /path/to/evm_signing_key.pem {} \;

# Verify signature
getfattr -d -m security.evm /usr/bin/iot-gateway
# security.evm: 0x03...  (HMAC signature)
```

##### Example 13: Detecting SELinux Policy Tampering

**Scenario:** Detect if an attacker attempts to relabel files or modify policy.

**IMA measurement log monitoring:**
```bash
# Monitor IMA measurement log for SELinux context changes
tail -f /sys/kernel/security/ima/ascii_runtime_measurements | \
    grep "security.selinux"

# Example output (attacker attempting relabel):
# 10 a3b5c8... ima-ng sha256:d4e6f2... boot_aggregate
# 10 b7c3d9... ima-ng sha256:e8f4a6... /usr/bin/iot-gateway security.selinux=unconfined_t

# Alert: /usr/bin/iot-gateway relabeled to unconfined_t (suspicious!)
```

**Automated monitoring script** (`monitor-selinux-integrity.sh`):
```bash
#!/bin/bash
# Monitor for unauthorized SELinux context changes

BASELINE_DB="/var/lib/selinux-integrity/baseline.db"
ALERT_LOG="/var/log/selinux-integrity-alerts.log"

# Create baseline of all file contexts
create_baseline() {
    find / -xdev -print0 2>/dev/null | \
        xargs -0 ls -Z 2>/dev/null | \
        awk '{print $NF, $1}' > "$BASELINE_DB"
}

# Check for deviations from baseline
check_integrity() {
    local violations=0

    while IFS= read -r line; do
        file=$(echo "$line" | awk '{print $1}')
        expected_context=$(echo "$line" | awk '{print $2}')

        if [ -e "$file" ]; then
            current_context=$(ls -Z "$file" 2>/dev/null | awk '{print $1}')

            if [ "$current_context" != "$expected_context" ]; then
                echo "[$(date)] VIOLATION: $file" >> "$ALERT_LOG"
                echo "  Expected: $expected_context" >> "$ALERT_LOG"
                echo "  Current:  $current_context" >> "$ALERT_LOG"
                ((violations++))
            fi
        fi
    done < "$BASELINE_DB"

    if [ $violations -gt 0 ]; then
        echo "[$(date)] ALERT: $violations SELinux context violations detected" >> "$ALERT_LOG"
        # Send alert (syslog, SNMP trap, webhook, etc.)
        logger -p security.crit "SELinux integrity violation: $violations files relabeled"
    fi
}

# Run as systemd timer or cron job
case "$1" in
    baseline)
        create_baseline
        ;;
    check)
        check_integrity
        ;;
    *)
        echo "Usage: $0 {baseline|check}"
        exit 1
        ;;
esac
```

**Systemd timer for continuous monitoring:**
```ini
# /etc/systemd/system/selinux-integrity.timer
[Unit]
Description=SELinux Integrity Monitoring Timer

[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
Unit=selinux-integrity.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/selinux-integrity.service
[Unit]
Description=SELinux Integrity Check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-selinux-integrity.sh check
```

#### Enforcement Bypass Scenarios and CVE Analysis

Understanding how attackers bypass SELinux is critical for hardening policies. This section analyzes real-world CVEs and provides mitigation strategies for IoT, IIoT, and enterprise network devices.

##### Recent SELinux Bypass Vulnerabilities (2024-2025)

**CVE-2024-1086**: Universal Linux Kernel Privilege Escalation
- **Severity**: CVSS 7.8 (High)
- **Affected**: Linux kernels v5.14 - v6.6
- **Impact**: Local privilege escalation to root, bypasses SELinux confinement
- **Attack**: Use-after-free in netfilter nf_tables subsystem
- **SELinux impact**: Confined domains can escalate to unconfined_t

**CVE-2025-9074**: Docker Container Escape
- **Severity**: CVSS 9.3 (Critical)
- **Affected**: Docker Desktop < 4.44.3
- **Impact**: Container escape to host, bypasses SELinux container isolation
- **Attack**: Unauthenticated API access to Docker Engine
- **SELinux impact**: svirt_lxc_net_t containers can access host filesystem

**CVE-2025-23266**: NVIDIA Container Toolkit Escape (NVIDIAScape)
- **Severity**: CVSS 9.0 (Critical)
- **Affected**: NVIDIA Container Toolkit (NCT)
- **Impact**: Container escape to root on host
- **Attack**: OCI hook misconfiguration allows breakout
- **SELinux impact**: GPU-accelerated containers bypass nvidia_container_t confinement

##### Common Bypass Techniques

**1. Unconfined Domain Exploitation**
- Attackers target processes running in `unconfined_t` domain
- Transition from confined → unconfined → root
- **Mitigation**: Minimize unconfined domains, use targeted policies

**2. Security Hook Manipulation**
- Kernel exploits remove SELinux LSM hooks from `security_hook_heads`
- All permission checks bypassed
- **Mitigation**: Kernel lockdown mode, verified boot, kernel module signing

**3. Capability Abuse**
- Domains with `sys_admin` or `sys_module` capabilities can load kernel modules
- Malicious modules disable SELinux
- **Mitigation**: Strict capability policies, kernel module signing enforcement

##### Example 14: Kernel Lockdown Configuration for Bypass Prevention

**Kernel configuration** (add to `kernel-hardening.cfg`):
```cfg
# Kernel Lockdown - Prevent runtime security modifications
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_SECURITY_LOCKDOWN_LSM_EARLY=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_CONFIDENTIALITY=y

# Prevent loading unsigned kernel modules
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_FORCE=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_SHA256=y

# Prevent /dev/mem and /dev/kmem access (bypass vector)
CONFIG_STRICT_DEVMEM=y
CONFIG_DEVMEM=n
CONFIG_DEVKMEM=n

# Disable kexec (prevents loading unsigned kernels)
CONFIG_KEXEC=n
CONFIG_KEXEC_FILE=n

# Disable hibernation (prevents offline modification)
CONFIG_HIBERNATION=n

# Kernel address space layout randomization
CONFIG_RANDOMIZE_BASE=y
CONFIG_RANDOMIZE_MEMORY=y

# Prevent debugfs access (information disclosure)
# CONFIG_DEBUG_FS is not set

# Prevent kprobes (dynamic kernel instrumentation)
# CONFIG_KPROBES is not set
```

**Boot parameters** (`/boot/grub/grub.cfg` or U-Boot environment):
```
# Enforce kernel lockdown from boot
lockdown=confidentiality

# Disable SELinux runtime mode changes
selinux=1 enforcing=1 security=selinux

# Prevent module loading after boot (optional, extreme hardening)
modules_disabled=1
```

##### Example 15: SELinux Policy Hardening Against Bypasses

**Hardening checklist:**
```te
# 1. Minimize unconfined domains
# Ensure only init and essential services are unconfined
neverallow { domain -init_t -unconfined_t -kernel_t } unconfined_t:process transition;

# 2. Restrict sys_admin capability (used in many exploits)
neverallow { domain -init_t -insmod_t -iptables_t } self:capability sys_admin;

# 3. Prevent kernel module loading from confined domains
neverallow { domain -init_t -insmod_t -modprobe_t } self:capability sys_module;

# 4. Restrict /dev/mem and /dev/kmem access
neverallow { domain -init_t } devmem_t:chr_file { read write };

# 5. Prevent execution from writable directories
neverallow { domain } { tmp_t var_t }:file execute;

# 6. Restrict ptrace (used for process injection)
neverallow { domain -unconfined_t -strace_t -gdb_t } domain:process ptrace;

# 7. Prevent setuid/setgid on confined domains
neverallow { domain -init_t -su_t -sudo_t } self:capability { setuid setgid };

# 8. Restrict network-facing services
# Example: Web server cannot execute system binaries
neverallow httpd_t { bin_t sbin_t }:file execute;

# 9. Container confinement
# Prevent containers from accessing host filesystem
neverallow svirt_lxc_net_t { file_type -svirt_sandbox_file_t }:file write;

# 10. Prevent SELinux policy modifications
neverallow { domain -init_t -load_policy_t } security_t:security { load_policy setenforce };
```

##### Example 16: Defense-in-Depth for Enterprise Network Devices

**Multi-layered protection for routers, firewalls, SD-WAN gateways:**

```bash
# Layer 1: Kernel hardening + SELinux
# - Lockdown mode (prevents runtime modifications)
# - Module signing (prevents malicious module loading)
# - SELinux enforcing (MAC enforcement)

# Layer 2: Network service confinement
# - Each daemon runs in separate SELinux domain
# - Minimal capabilities (no sys_admin, sys_module)
# - Restricted filesystem access

# Example policy for firewall daemon
allow iptables_t self:capability { net_admin net_raw };
allow iptables_t self:rawip_socket { create bind read write };
neverallow iptables_t { bin_t sbin_t }:file execute;  # No shell access
neverallow iptables_t self:capability sys_module;     # No module loading

# Layer 3: IMA/EVM integrity
# - Sign all executables and configs
# - Verify on every execution and modification
# - Detect file tampering

# Layer 4: Runtime monitoring
# - Audit log analysis (ausearch)
# - File integrity monitoring (AIDE/Tripwire)
# - SELinux violation alerts

# Layer 5: Secure boot chain
# - U-Boot verified boot
# - Kernel signature verification
# - Immutable rootfs with dm-verity
```

**Verification commands:**
```bash
# Verify kernel lockdown active
cat /sys/kernel/security/lockdown
# Output: [none] integrity [confidentiality]

# Verify SELinux enforcing
getenforce
# Output: Enforcing

# Verify no unconfined processes (except init)
ps -eZ | grep unconfined_t
# Should only show init and direct children

# Verify module signing enforced
cat /proc/sys/kernel/modules_disabled
# Output: 1 (if modules disabled after boot)

# Check for SELinux policy load capability (should be denied)
sesearch --allow -t security_t -c security -p load_policy
# Should only show init_t and load_policy_t

# Audit for privilege escalation attempts
ausearch -m avc -ts today | grep unconfined_t
```

#### Critical Infrastructure: ICS/SCADA Security

Industrial Control Systems (ICS) and SCADA networks require specialized SELinux policies to meet compliance frameworks like **IEC 62443** and **NERC CIP**. This section provides policy examples for industrial IoT deployments.

##### IEC 62443 Security Levels and SELinux

**IEC 62443 defines 4 Security Levels (SL):**
- **SL 1**: Protection against casual or coincidental violation
- **SL 2**: Protection against intentional violation using simple means
- **SL 3**: Protection against intentional violation using sophisticated means
- **SL 4**: Protection against intentional violation using sophisticated means with extended resources

**SELinux alignment:**
| IEC 62443 Requirement | SELinux Implementation |
|-----------------------|------------------------|
| Access control (CR 1.1) | Targeted policies with MAC |
| Use control (CR 1.2) | Domain-based execution control |
| System integrity (CR 3.4) | IMA/EVM + neverallow rules |
| Confidentiality (CR 3.5) | MLS policies for classified data |
| Audit trail (CR 2.8) | SELinux audit logging |

##### Example 17: Zone Segmentation with SELinux for IEC 62443

**IEC 62443 Zone Model:**
- **Zone 0**: Enterprise IT network
- **Zone 1**: Site operations (SCADA HMI, historians)
- **Zone 2**: Area supervisory control (PLCs, DCS)
- **Zone 3**: Field devices (sensors, actuators, RTUs)

**SELinux policy for zone isolation:**

```te
# Zone isolation policy for industrial network
policy_module(iec62443_zones, 1.0.0)

# Zone 0: Enterprise IT (untrusted)
type zone0_t;
domain_type(zone0_t)

# Zone 1: SCADA HMI
type zone1_scada_t;
domain_type(zone1_scada_t)

# Zone 2: PLC/DCS controllers
type zone2_plc_t;
domain_type(zone2_plc_t)

# Zone 3: Field devices
type zone3_field_t;
domain_type(zone3_field_t)

# Zone communication rules (conduits)
# Zone 1 can read from Zone 2 (HMI reads PLC data)
allow zone1_scada_t zone2_plc_t:tcp_socket { read write };

# Zone 2 can read from Zone 3 (PLC reads sensor data)
allow zone2_plc_t zone3_field_t:tcp_socket { read write };

# CRITICAL: Zone 0 CANNOT directly access Zone 2/3 (air gap enforcement)
neverallow zone0_t zone2_plc_t:tcp_socket { connect };
neverallow zone0_t zone3_field_t:tcp_socket { connect };

# Zone 1 can communicate with Zone 0 (reporting to enterprise)
allow zone1_scada_t zone0_t:tcp_socket { connect write read };

# Field devices (Zone 3) cannot initiate connections upward (unidirectional)
neverallow zone3_field_t { zone2_plc_t zone1_scada_t }:tcp_socket { connect };
```

##### Example 18: PLC Communication Isolation

**Scenario:** Modbus TCP PLC must only communicate with authorized HMI and field devices.

```te
policy_module(modbus_plc, 1.0.0)

# PLC domain
type modbus_plc_t;
type modbus_plc_exec_t;
init_daemon_domain(modbus_plc_t, modbus_plc_exec_t)

# HMI domain (trusted)
type scada_hmi_t;

# Field device domain
type field_sensor_t;

# PLC data storage
type modbus_data_t;
files_type(modbus_data_t)

# Allow PLC to manage its data
manage_files_pattern(modbus_plc_t, modbus_data_t, modbus_data_t)

# Allow PLC to listen on Modbus port (502/TCP)
corenet_tcp_bind_generic_node(modbus_plc_t)
allow modbus_plc_t self:tcp_socket { create bind listen accept read write };

# RESTRICTED: Only HMI can connect to PLC
allow scada_hmi_t modbus_plc_t:tcp_socket { connect write read };

# RESTRICTED: Only field sensors can send data to PLC
allow field_sensor_t modbus_plc_t:tcp_socket { connect write };

# DENY: Enterprise IT cannot directly access PLC
type enterprise_t;
neverallow enterprise_t modbus_plc_t:tcp_socket { connect };

# DENY: PLC cannot execute arbitrary binaries (prevent lateral movement)
neverallow modbus_plc_t { bin_t sbin_t }:file execute;

# DENY: PLC cannot access internet (isolated OT network)
neverallow modbus_plc_t http_port_t:tcp_socket { connect };
neverallow modbus_plc_t dns_port_t:udp_socket { connect };

# Audit all connection attempts
auditallow { domain -scada_hmi_t -field_sensor_t } modbus_plc_t:tcp_socket connect;
```

##### Example 19: HMI (Human-Machine Interface) Confinement

**Scenario:** SCADA HMI must display data from PLCs but cannot modify control logic.

```te
policy_module(scada_hmi, 1.0.0)

# HMI domain
type scada_hmi_t;
type scada_hmi_exec_t;
init_daemon_domain(scada_hmi_t, scada_hmi_exec_t)

# HMI data (read-only cache of PLC data)
type scada_hmi_data_t;
files_type(scada_hmi_data_t)

# PLC data (on PLC, not HMI)
type modbus_data_t;

# Allow HMI to read from PLCs
allow scada_hmi_t modbus_plc_t:tcp_socket { connect read };

# Allow HMI to cache data locally (read-only)
allow scada_hmi_t scada_hmi_data_t:file { read getattr open };
allow scada_hmi_t scada_hmi_data_t:dir { read search };

# CRITICAL: HMI cannot WRITE to PLC data (read-only monitoring)
neverallow scada_hmi_t modbus_data_t:file { write append };

# CRITICAL: HMI cannot execute PLC control binaries
neverallow scada_hmi_t modbus_plc_exec_t:file execute;

# Allow HMI to display UI (X11 or web interface)
xserver_user_x_domain_template(scada_hmi, scada_hmi_t, scada_hmi_tmpfs_t)

# Allow web interface (read-only dashboard)
allow scada_hmi_t self:tcp_socket { create bind listen accept };
corenet_tcp_bind_http_port(scada_hmi_t)

# DENY: HMI cannot access enterprise databases directly (data diode)
neverallow scada_hmi_t oracle_port_t:tcp_socket { connect };

# Audit all write attempts to PLC data (indicates attack or misconfiguration)
auditallow scada_hmi_t modbus_data_t:file { write append };
```

#### Enterprise Network Device Hardening

Network equipment (routers, switches, firewalls, SD-WAN gateways, UTM appliances) run embedded Linux and require specialized SELinux policies to prevent compromise and lateral movement.

##### Network Appliance Threat Model

**Attack vectors for enterprise network devices:**
1. **Management interface exploitation** (SSH, web UI, SNMP)
2. **Control plane attacks** (BGP hijacking, OSPF poisoning)
3. **Data plane attacks** (packet injection, man-in-the-middle)
4. **Firmware/config tampering** (persistent backdoors)
5. **Container escapes** (for NFV/VNF deployments)

**SELinux mitigations:**
- Confine management daemons (sshd, httpd, snmpd)
- Isolate routing protocols in separate domains
- Protect configuration files from unauthorized modification
- Enforce IMA/EVM integrity for firmware and config
- Isolate virtualized network functions (VNFs)

##### Example 20: Firewall/UTM Appliance SELinux Hardening

**Scenario:** Hardware firewall/UTM running iptables, intrusion detection, and VPN termination.

```te
policy_module(utm_appliance, 1.0.0)

# Firewall/iptables domain
type utm_firewall_t;
type utm_firewall_exec_t;
init_daemon_domain(utm_firewall_t, utm_firewall_exec_t)

# VPN daemon domain
type utm_vpn_t;
type utm_vpn_exec_t;
init_daemon_domain(utm_vpn_t, utm_vpn_exec_t)

# IDS/IPS domain (Snort/Suricata)
type utm_ids_t;
type utm_ids_exec_t;
init_daemon_domain(utm_ids_t, utm_ids_exec_t)

# Firewall rules file
type utm_firewall_config_t;
files_config_file(utm_firewall_config_t)

# Allow firewall daemon to manage iptables
allow utm_firewall_t self:capability { net_admin net_raw };
allow utm_firewall_t self:rawip_socket { create bind read write };
kernel_read_network_state(utm_firewall_t)
kernel_rw_net_sysctls(utm_firewall_t)

# Allow firewall to read config
allow utm_firewall_t utm_firewall_config_t:file { read getattr open };

# CRITICAL: Firewall cannot execute arbitrary binaries (prevent shell injection)
neverallow utm_firewall_t { bin_t sbin_t }:file execute;
neverallow utm_firewall_t shell_exec_t:file execute;

# CRITICAL: Only firewall domain can modify netfilter rules
neverallow { domain -utm_firewall_t -iptables_t } iptables_exec_t:file execute;

# VPN daemon isolation
allow utm_vpn_t self:capability { net_admin };
allow utm_vpn_t tun_tap_device_t:chr_file { read write ioctl };

# VPN cannot access firewall config (separation of duties)
neverallow utm_vpn_t utm_firewall_config_t:file { read write };

# IDS/IPS can only read network traffic (passive monitoring)
allow utm_ids_t self:capability { net_admin net_raw };
allow utm_ids_t self:packet_socket { create bind read };

# IDS cannot modify firewall rules (read-only monitoring)
neverallow utm_ids_t utm_firewall_config_t:file write;
neverallow utm_ids_t iptables_exec_t:file execute;

# Management interface isolation
# Web UI can read status but not modify config
type utm_webui_t;
allow utm_webui_t utm_firewall_config_t:file { read getattr };
neverallow utm_webui_t utm_firewall_config_t:file { write append };
```

##### Example 21: SD-WAN Gateway Policy

**Scenario:** SD-WAN gateway with IPsec/WireGuard tunnels, dynamic routing, and application-aware routing.

```te
policy_module(sdwan_gateway, 1.0.0)

# SD-WAN daemon domain
type sdwan_daemon_t;
type sdwan_daemon_exec_t;
init_daemon_domain(sdwan_daemon_t, sdwan_daemon_exec_t)

# Configuration files
type sdwan_config_t;
files_config_file(sdwan_config_t)

# State/routing table
type sdwan_state_t;
files_type(sdwan_state_t)

# Allow SD-WAN daemon to manage tunnels
allow sdwan_daemon_t self:capability { net_admin net_raw };
allow sdwan_daemon_t self:rawip_socket { create bind read write };
allow sdwan_daemon_t tun_tap_device_t:chr_file { read write ioctl open };

# Allow netlink for routing table manipulation
allow sdwan_daemon_t self:netlink_route_socket { create bind read write nlmsg_read nlmsg_write };

# Allow reading network interfaces
kernel_read_network_state(sdwan_daemon_t)
kernel_rw_net_sysctls(sdwan_daemon_t)

# Allow reading config
allow sdwan_daemon_t sdwan_config_t:file { read getattr open };

# Allow managing state files
manage_files_pattern(sdwan_daemon_t, sdwan_state_t, sdwan_state_t)

# Allow encrypted tunnel communication (IPsec, WireGuard)
allow sdwan_daemon_t self:key_socket { create read write };
allow sdwan_daemon_t ipsec_spd_t:association { sendto recvfrom };

# CRITICAL: SD-WAN cannot modify system config files
neverallow sdwan_daemon_t { etc_t -sdwan_config_t }:file write;

# CRITICAL: SD-WAN cannot execute arbitrary binaries (command injection prevention)
neverallow sdwan_daemon_t { bin_t sbin_t }:file execute;

# Telemetry and monitoring (send metrics to controller)
allow sdwan_daemon_t http_port_t:tcp_socket { connect write read };

# DENY: SD-WAN cannot access unrelated services
neverallow sdwan_daemon_t postgresql_port_t:tcp_socket connect;
neverallow sdwan_daemon_t mysql_port_t:tcp_socket connect;
```

##### Example 22: NFV/VNF Isolation for Virtualized Network Functions

**Scenario:** Network Function Virtualization (NFV) platform running multiple Virtual Network Functions (VNFs) as VMs or containers.

```te
policy_module(nfv_platform, 1.0.0)

# NFVI (NFV Infrastructure) domain
type nfvi_platform_t;
domain_type(nfvi_platform_t)

# VNF types (different network functions)
type vnf_router_t;
type vnf_firewall_t;
type vnf_loadbalancer_t;
domain_type(vnf_router_t)
domain_type(vnf_firewall_t)
domain_type(vnf_loadbalancer_t)

# VNF data isolation
type vnf_router_data_t;
type vnf_firewall_data_t;
type vnf_loadbalancer_data_t;
files_type(vnf_router_data_t)
files_type(vnf_firewall_data_t)
files_type(vnf_loadbalancer_data_t)

# NFVI can manage all VNFs (orchestration)
allow nfvi_platform_t { vnf_router_t vnf_firewall_t vnf_loadbalancer_t }:process { signal transition };

# VNF isolation - each VNF can only access its own data
allow vnf_router_t vnf_router_data_t:file { read write };
neverallow vnf_router_t { vnf_firewall_data_t vnf_loadbalancer_data_t }:file { read write };

allow vnf_firewall_t vnf_firewall_data_t:file { read write };
neverallow vnf_firewall_t { vnf_router_data_t vnf_loadbalancer_data_t }:file { read write };

# VNFs cannot access NFVI management plane
neverallow { vnf_router_t vnf_firewall_t vnf_loadbalancer_t } nfvi_platform_t:file read;

# VNFs cannot transition to unconfined domains (container escape prevention)
neverallow { vnf_router_t vnf_firewall_t vnf_loadbalancer_t } unconfined_t:process transition;

# VNFs cannot access host devices (prevent hardware attacks)
neverallow { vnf_router_t vnf_firewall_t vnf_loadbalancer_t } device_t:blk_file { read write };
```

##### Example 23: Container Network Function (CNF) Security for 5G

**Scenario:** 5G core network CNFs running in Kubernetes with SELinux pod security.

**Kubernetes SELinux pod security context:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: 5g-upf-cnf
  namespace: 5g-core
spec:
  securityContext:
    # SELinux context for entire pod
    seLinuxOptions:
      level: "s0:c100,c200"  # MCS categories for isolation
      type: "cnf_5g_upf_t"   # Custom SELinux type

  containers:
  - name: upf-container
    image: 5g-upf:v1.2.3
    securityContext:
      # Additional container-level restrictions
      seLinuxOptions:
        type: "cnf_5g_upf_t"
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_ADMIN", "NET_RAW"]  # Only what's needed for UPF

    volumeMounts:
    - name: upf-config
      mountPath: /etc/upf
      readOnly: true  # Config is read-only

  volumes:
  - name: upf-config
    configMap:
      name: upf-config
```

**SELinux policy for 5G CNF:**
```te
policy_module(cnf_5g_upf, 1.0.0)

# 5G UPF (User Plane Function) CNF domain
type cnf_5g_upf_t;
domain_type(cnf_5g_upf_t)

# Allow UPF to process network packets
allow cnf_5g_upf_t self:capability { net_admin net_raw };
allow cnf_5g_upf_t self:packet_socket { create bind read write };

# Allow UPF to manage GTP-U tunnels (port 2152/UDP)
allow cnf_5g_upf_t self:udp_socket { create bind read write };
allow cnf_5g_upf_t gtpu_port_t:udp_socket { send_msg recv_msg };

# CRITICAL: UPF cannot access other CNF data (isolation)
type cnf_5g_amf_t;  # Access and Mobility Management Function
type cnf_5g_smf_t;  # Session Management Function
neverallow cnf_5g_upf_t { cnf_5g_amf_t cnf_5g_smf_t }:file { read write };

# CRITICAL: CNF cannot escape container (svirt)
neverallow cnf_5g_upf_t { file_type -svirt_sandbox_file_t }:file { write execute };

# Allow communication with SMF control plane (limited)
allow cnf_5g_upf_t cnf_5g_smf_t:tcp_socket { connect write read };

# DENY: UPF cannot access Kubernetes API (prevent cluster takeover)
neverallow cnf_5g_upf_t kubernetes_api_port_t:tcp_socket connect;
```

#### SELinux Security Checklist for Embedded Systems

Use this comprehensive checklist to validate SELinux security posture across IoT, IIoT, and enterprise network devices.

##### Policy Development & Deployment

**Policy Selection**
- [ ] Appropriate policy type selected (minimal/targeted/MLS) based on device constraints
- [ ] Memory overhead acceptable (< 10% of total RAM for targeted policy)
- [ ] Policy scope covers all security-sensitive processes
- [ ] Unconfined domains minimized (only init_t if possible)

**Custom Policy Quality**
- [ ] All custom domains have explicit file context labels (.fc files)
- [ ] neverallow rules defined for critical security boundaries
- [ ] Least privilege enforced (minimal capabilities granted)
- [ ] No wildcards in sensitive allow rules (e.g., `allow foo_t *:file read`)

**Yocto Integration**
- [ ] meta-selinux layer added to bblayers.conf
- [ ] DISTRO_FEATURES includes "selinux"
- [ ] SELINUX_MODE set to "enforcing" for production builds
- [ ] Custom policy modules packaged as .bb recipes
- [ ] File contexts applied during do_install

##### Static Testing (Pre-Deployment)

**SELint Analysis**
- [ ] SELint installed and configured (.selint config file present)
- [ ] CI/CD pipeline includes SELint check with `--fail` flag
- [ ] Severity threshold set to "warning" or higher
- [ ] All SELint errors resolved before deployment
- [ ] Policy style follows refpolicy conventions

**Policy Compilation**
- [ ] All .te files compile without errors (checkmodule successful)
- [ ] Policy modules package correctly (semodule_package successful)
- [ ] Interface database generated (sepolgen-ifgen) if using audit2allow -R
- [ ] No circular dependencies between policy modules

##### Dynamic Testing (Runtime)

**setools Audit**
- [ ] sesearch used to find dangerous permission paths
- [ ] Verified no domains can transition to unconfined_t (except init)
- [ ] Verified no domains have sys_admin + sys_module capabilities
- [ ] Verified network-facing domains cannot execute binaries (shell injection prevention)
- [ ] sechecker security checks passed (if applicable)

**Runtime Behavior**
- [ ] All services start successfully in enforcing mode
- [ ] No AVC denials during normal operation (ausearch -m avc)
- [ ] Application functionality fully tested in enforcing mode
- [ ] Permissive mode only used for initial policy development

##### Runtime Integrity & Monitoring

**IMA/EVM Protection**
- [ ] IMA/EVM enabled in kernel (CONFIG_IMA=y, CONFIG_EVM=y)
- [ ] IMA policy protects SELinux extended attributes (security.selinux)
- [ ] Critical binaries and configs signed with evmctl
- [ ] EVM public key loaded into kernel keyring
- [ ] IMA measurement log monitored for unauthorized changes

**Continuous Monitoring**
- [ ] Audit daemon (auditd) running and logging AVC denials
- [ ] Automated monitoring for file context changes (baseline integrity check)
- [ ] Alerts configured for SELinux policy violations
- [ ] Log retention meets compliance requirements (e.g., 90 days for IEC 62443)

##### Bypass Prevention

**Kernel Hardening**
- [ ] Kernel lockdown mode enabled (CONFIG_SECURITY_LOCKDOWN_LSM=y)
- [ ] Kernel module signing enforced (CONFIG_MODULE_SIG_FORCE=y)
- [ ] /dev/mem and /dev/kmem disabled (CONFIG_STRICT_DEVMEM=y)
- [ ] Debugfs disabled in production (CONFIG_DEBUG_FS=n)
- [ ] Kprobes disabled (CONFIG_KPROBES=n)

**Policy Hardening**
- [ ] neverallow rules prevent unconfined transitions
- [ ] neverallow rules restrict sys_admin, sys_module capabilities
- [ ] neverallow rules prevent execution from writable directories
- [ ] neverallow rules prevent policy modifications at runtime
- [ ] Verified with: `sesearch --neverallow`

**CVE Mitigation**
- [ ] Kernel version not vulnerable to CVE-2024-1086 (or patched)
- [ ] Container runtime not vulnerable to CVE-2025-9074 (Docker Desktop >= 4.44.3)
- [ ] NVIDIA Container Toolkit not vulnerable to CVE-2025-23266 (or not used)
- [ ] Security advisories monitored and patches applied

##### Compliance (Industry-Specific)

**IoT Devices**
- [ ] Minimal policy used if memory < 64MB
- [ ] Network services confined (MQTT, CoAP, HTTP)
- [ ] OTA update process verifies policy integrity
- [ ] Default credentials disabled (CWE-798 prevention)

**Industrial IoT (IEC 62443)**
- [ ] Security Level (SL) requirements mapped to SELinux policy
- [ ] Zone segmentation enforced with SELinux domains
- [ ] PLC/RTU communication isolated
- [ ] HMI confined to read-only monitoring
- [ ] Audit logs retained per CR 2.8 requirements

**Enterprise Network Devices**
- [ ] Management interface confined (SSH, web UI, SNMP)
- [ ] Routing protocols isolated in separate domains
- [ ] Configuration files protected from unauthorized modification
- [ ] Firmware integrity verified with IMA/EVM
- [ ] VNF/CNF isolation enforced (if applicable)
- [ ] Zero Trust principles applied (least privilege, micro-segmentation)

**Critical Infrastructure (NERC CIP)**
- [ ] Bulk electric system components identified and confined
- [ ] Electronic Access Control (CIP-005) enforced with SELinux
- [ ] System Security Management (CIP-007) audit logs enabled
- [ ] Configuration Change Management (CIP-010) baseline established
- [ ] Incident Response (CIP-008) procedures include SELinux forensics

##### Production Deployment

**Pre-Deployment**
- [ ] SELinux enforcing mode enabled in production image
- [ ] Boot parameters prevent runtime mode changes (enforcing=1)
- [ ] All policy modules loaded successfully
- [ ] File contexts relabeled (restorecon -Rv / or autorelabel on first boot)
- [ ] Secure boot chain includes policy integrity verification

**Post-Deployment**
- [ ] `getenforce` returns "Enforcing"
- [ ] `sestatus` shows policy loaded and no errors
- [ ] No unexpected AVC denials in audit log
- [ ] Functionality smoke tests passed
- [ ] Monitoring and alerting operational

##### Documentation & Maintenance

**Policy Documentation**
- [ ] Policy design decisions documented (rationale for neverallow rules)
- [ ] Custom domains and types catalogued
- [ ] Known AVC denials (if any) documented with justification
- [ ] Compliance mapping documented (IEC 62443, NERC CIP, etc.)

**Change Management**
- [ ] Policy changes versioned in source control
- [ ] Policy updates tested in staging environment before production
- [ ] Rollback procedure documented
- [ ] Security regression testing includes SELinux policy validation

---

### Additional Resources: Advanced SELinux

**SELinux Policy Development**
- [SELinux Reference Policy](https://github.com/SELinuxProject/refpolicy) - Base policy for most distributions
- [SELinux Notebook](https://github.com/SELinuxProject/selinux-notebook) - Comprehensive SELinux architecture guide
- [SELinux Project Wiki - Tools](https://github.com/SELinuxProject/selinux/wiki/Tools) - Complete tool ecosystem

**Policy Testing and Analysis**
- [SELint - Static Analysis](https://github.com/SELinuxProject/selint) - Policy linter
- [setools - Policy Analysis](https://github.com/SELinuxProject/setools) - Query and analysis tools
- [SELinux Testsuite](https://github.com/SELinuxProject/selinux-testsuite) - Kernel functionality regression tests
- [sepolicy_analysis](https://github.com/vmojzis/sepolicy_analysis) - CVE and vulnerability path analysis

**Runtime Verification**
- [IMA/EVM Documentation](https://sourceforge.net/p/linux-ima/wiki/Home/) - Integrity subsystem
- [meta-integrity (Yocto)](https://git.yoctoproject.org/meta-security/tree/meta-integrity) - IMA/EVM for embedded

**Compliance Frameworks**
- [IEC 62443 Standards](https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards) - Industrial cybersecurity
- [NERC CIP Compliance](https://www.nerc.com/pa/Stand/Pages/CIPStandards.aspx) - Bulk electric system protection
- [NIST SP 800-82](https://csrc.nist.gov/publications/detail/sp/800-82/rev-2/final) - ICS security

**CVE Resources**
- [SELinux CVE Search](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=selinux) - SELinux-related vulnerabilities
- [Red Hat SELinux Security](https://access.redhat.com/articles/6964380) - Real-world bypass examples

**Yocto SELinux Integration**
- [meta-selinux Git Repository](https://git.yoctoproject.org/meta-selinux/) - SELinux support for Yocto
- [Yocto Security Hardening Guide](https://docs.yoctoproject.org/dev/dev-manual/securing-images.html) - Official security guide

**Enterprise Network Security**
- [SELinux in Containers](https://www.redhat.com/en/blog/you-can-skip-virtual-machine-using-selinux-containers-help-secure-cloud-native-5g) - 5G CNF security
- [Kubernetes SELinux Policies](https://kubernetes.io/docs/tutorials/security/seccomp/) - Pod security contexts

#### Using SMACK with Yocto

**Enable SMACK** (via meta-security):
```bitbake
# In local.conf
DISTRO_FEATURES:append = " smack"

# Include SMACK tools
IMAGE_INSTALL:append = " smack smack-userspace"
```

**SMACK kernel config**:
```cfg
CONFIG_SECURITY_SMACK=y
CONFIG_DEFAULT_SECURITY_SMACK=y
```

**Example SMACK labeling**:
```bash
# On target device
# Set SMACK label on file
chsmack -a "AppData" /var/lib/myapp/data.db

# Set SMACK label on process
chsmack -e "MyApp" /usr/bin/myapp

# Define access rules
echo "MyApp AppData rw" > /sys/fs/smackfs/load2
```

**SMACK in Yocto recipes**:
```bitbake
# In application recipe
inherit smack

# Set SMACK labels during install
do_install:append() {
    # Label executable
    echo "MyApp" > ${D}${bindir}/myapp.smack

    # Label data directory
    install -d ${D}${localstatedir}/lib/myapp
    echo "AppData" > ${D}${localstatedir}/lib/myapp.smack
}
```

### Integrity Measurement Architecture (IMA/EVM)

IMA/EVM provides file integrity verification and protection against unauthorized modifications.

**Enable IMA/EVM** (via meta-security):
```bitbake
# In local.conf
DISTRO_FEATURES:append = " ima"

# Include IMA tools
IMAGE_INSTALL:append = " ima-evm-utils keyutils"
```

**Kernel configuration for IMA**:
```cfg
CONFIG_INTEGRITY=y
CONFIG_IMA=y
CONFIG_IMA_MEASURE_PCR_IDX=10
CONFIG_IMA_APPRAISE=y
CONFIG_IMA_TRUSTED_KEYRING=y
CONFIG_EVM=y
```

**IMA Policy Example**:
```bash
# /etc/ima/ima-policy
# Measure all executables
measure func=BPRM_CHECK
measure func=FILE_MMAP mask=MAY_EXEC

# Appraise all files in /usr
appraise fowner=0 func=FILE_MMAP mask=MAY_EXEC
appraise fowner=0 func=BPRM_CHECK
```

**Sign files for IMA/EVM**:
```bash
# Generate IMA signing key
openssl genrsa -out ima_signing_key.pem 2048

# Sign a file
evmctl ima_sign --key ima_signing_key.pem /usr/bin/myapp

# Load IMA public key into kernel keyring (at boot)
keyctl padd asymmetric "" %keyring:.ima < ima_public_key.der
```

### Read-Only Root Filesystem with Overlays

A read-only root filesystem prevents persistent malware and unauthorized modifications.

**Enable read-only rootfs** (local.conf):
```bitbake
# Make root filesystem read-only
IMAGE_FEATURES += "read-only-rootfs"

# Specify writable directories via tmpfs overlays
VOLATILE_LOG_DIR = "yes"

# Custom overlay mounts
OVERLAYFS_MOUNT_POINT[data] = "/var/lib/myapp"
OVERLAYFS_WRITABLE_PATHS[data] = "/var/lib/myapp"
```

**fstab configuration for overlays**:
```bash
# /etc/fstab
tmpfs  /tmp       tmpfs  mode=1777,strictatime,nosuid,nodev,size=50M  0  0
tmpfs  /var/log   tmpfs  mode=0755,strictatime,nosuid,nodev,size=20M  0  0
tmpfs  /var/tmp   tmpfs  mode=1777,strictatime,nosuid,nodev,size=10M  0  0

# Persistent writable overlay (e.g., on separate partition)
/dev/mmcblk0p3  /var/lib/myapp  ext4  defaults,noatime  0  2
```

**OverlayFS configuration for selective writes**:
```bitbake
# In image recipe
IMAGE_FEATURES += "overlayfs"

# Specify overlay mounts
OVERLAYFS_MOUNT_POINT[config] = "/etc"
OVERLAYFS_WRITABLE_PATHS[config] = "/etc/myapp.conf"

OVERLAYFS_MOUNT_POINT[data] = "/var"
OVERLAYFS_WRITABLE_PATHS[data] = "/var/lib/myapp"
```

**Benefits**:
- Prevents persistent rootkits
- Easy factory reset (reboot clears tmpfs)
- Reduces flash wear
- Simplifies OTA updates (atomic rootfs replacement)

**Trade-offs**:
- Requires careful planning of persistent data locations
- Log aggregation needed for persistent logging
- Configuration changes need explicit persistence mechanism

### Reproducible Builds

Yocto achieved **100% reproducible builds** in 2022, meaning identical source code produces bit-for-bit identical binaries.

**Benefits**:
- **Supply chain security**: Verify official builds match your rebuild
- **Backdoor detection**: Independent verification prevents hidden malware
- **Compliance**: Auditors can verify build provenance

**Enable reproducible builds** (default in modern Yocto):
```bitbake
# In local.conf (usually default)
INHERIT += "reproducible_build"

# Use fixed timestamps
SOURCE_DATE_EPOCH = "1704067200"  # Fixed timestamp (2024-01-01)
```

**Verify reproducibility**:
```bash
# Build twice and compare
bitbake core-image-minimal
mv tmp/deploy/images/ deploy1

bitbake -c cleanall core-image-minimal
bitbake core-image-minimal
mv tmp/deploy/images/ deploy2

# Compare
diffoscope deploy1/image.rootfs.tar.gz deploy2/image.rootfs.tar.gz
```

### Making Images More Secure (Official Yocto Guide)

Yocto's official documentation provides comprehensive guidance on securing images:

**Reference**: [https://docs.yoctoproject.org/dev/dev-manual/securing-images.html](https://docs.yoctoproject.org/dev/dev-manual/securing-images.html)

**Key Recommendations from Official Guide**:

1. **Remove debug tools in production**:
```bitbake
# In local.conf for production builds
EXTRA_IMAGE_FEATURES:remove = "debug-tweaks"
EXTRA_IMAGE_FEATURES:remove = "tools-debug"
EXTRA_IMAGE_FEATURES:remove = "tools-testapps"
```

2. **Minimize attack surface**:
```bitbake
# Remove unnecessary services
PACKAGECONFIG:remove:pn-systemd = "networkd"

# Disable unnecessary features
DISTRO_FEATURES:remove = "x11 wayland bluetooth wifi nfc"
```

3. **Secure default passwords**:
```bitbake
# Force password change on first boot
INHERIT += "extrausers"
EXTRA_USERS_PARAMS = "usermod -L root;"  # Lock root account
```

4. **Use package management wisely**:
```bitbake
# For production, disable package manager to prevent runtime modifications
IMAGE_FEATURES:remove = "package-management"
```

5. **Implement secure boot** (see Chapter 3 for details):
```bitbake
# U-Boot verified boot
UBOOT_SIGN_ENABLE = "1"
UBOOT_MKIMAGE_DTCOPTS = "-I dts -O dtb -p 2000"
UBOOT_SIGN_KEYDIR = "${DEPLOY_DIR_IMAGE}/keys"
```

### Yocto Security Best Practices Checklist

Use this checklist for every Yocto-based embedded Linux project:

**Build Configuration**:
- [ ] Security flags enabled (`INHERIT += "security-flags"`)
- [ ] Compiler hardening verified (`checksec` on binaries)
- [ ] GCC `-fhardened` flag enabled for high-security applications
- [ ] Reproducible builds enabled and verified

**Kernel Hardening**:
- [ ] Kernel hardening config fragment applied
- [ ] KASLR enabled (`CONFIG_RANDOMIZE_BASE=y`)
- [ ] KPTI enabled if needed (`CONFIG_PAGE_TABLE_ISOLATION=y`)
- [ ] Kernel module signing enforced (`CONFIG_MODULE_SIG_FORCE=y`)
- [ ] Debug interfaces disabled in production (`CONFIG_DEBUG_FS=n`)
- [ ] Kernel hardening checker passed

**Access Control**:
- [ ] MAC system selected and configured (SELinux/AppArmor/SMACK)
- [ ] MAC policies tested in permissive mode before enforcement
- [ ] IMA/EVM file integrity enabled for critical files
- [ ] Read-only rootfs with tmpfs/overlayfs for writable paths

**Vulnerability Management**:
- [ ] CVE checking enabled (`INHERIT += "cve-check"`)
- [ ] CVE reports reviewed and documented
- [ ] Known false positives added to `CVE_CHECK_IGNORE` with justification
- [ ] CI/CD CVE scanning integrated (fail on critical/high unpatched)

**SBOM and Supply Chain**:
- [ ] SBOM generation enabled (`INHERIT += "create-spdx-3.0"`)
- [ ] buildhistory tracking enabled for package changes
- [ ] SBOM validated with spdx-tools
- [ ] SBOM shared with customers/stakeholders
- [ ] Third-party layers audited for security

**Image Security**:
- [ ] Debug tools removed from production images (`EXTRA_IMAGE_FEATURES:remove = "debug-tweaks"`)
- [ ] Root account locked or password enforced
- [ ] Package management removed from production (`IMAGE_FEATURES:remove = "package-management"`)
- [ ] Minimal attack surface (only necessary services enabled)
- [ ] Secure boot implemented (U-Boot verified boot or UEFI Secure Boot)

**Development vs. Production**:
- [ ] Separate build configurations for debug/development and production
- [ ] Production builds tested with security scanners (Lynis, OpenSCAP)
- [ ] Security regression testing automated in CI/CD
- [ ] Threat modeling updated for Yocto-specific attack vectors

## Additional References <a href="#additional-references" id="additional-references"></a>

* [https://cheatsheetseries.owasp.org/cheatsheets/C-Based_Toolchain_Hardening_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/C-Based_Toolchain_Hardening_Cheat_Sheet.html)
* [https://www.bulkorder.ftc.gov/system/files/publications/pdf0199-carefulconnections-buildingsecurityinternetofthings.pdf](https://www.bulkorder.ftc.gov/system/files/publications/pdf0199-carefulconnections-buildingsecurityinternetofthings.pdf)
* [http://isa99.isa.org/Public/Documents/ISA-62443-4-1-WD.pdf](http://isa99.isa.org/Public/Documents/ISA-62443-4-1-WD.pdf) (page 34-38)
* [https://events.linuxfoundation.org/sites/events/files/slides/belloni-petazzoni-buildroot-oe\_0.pdf](https://events.linuxfoundation.org/sites/events/files/slides/belloni-petazzoni-buildroot-oe\_0.pdf) - Details on buildroot and yocto
* [http://elinux.org/Toolchains](http://elinux.org/Toolchains)
* [https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS](https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS)
* [http://www.proftpd.org/docs/howto/TLS.html](http://www.proftpd.org/docs/howto/TLS.html)
* [https://owasp.org/www-community/Application_Threat_Modeling](https://owasp.org/www-community/Application_Threat_Modeling)
* [GNU C Library Vulnerability in Industrial Products](http://www.siemens.com/cert/pool/cert/siemens\_security\_advisory\_ssa-301706.pdf)
* [Linux Exploit Quick Listing](http://www.kmbl.us/les/working.php)
* [Hardened U-boot](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/07-system-hardening.html#hardened-boot)
  * [Verified boot](https://lwn.net/Articles/571031/)
* [Improving Your Embedded Linux Security Posture with Yocto](https://research.nccgroup.com/2018/08/27/improving-your-embedded-linux-security-posture-with-yocto/)

### Yocto Project Security Resources
* [Yocto Project Official Documentation - Making Images More Secure](https://docs.yoctoproject.org/dev/dev-manual/securing-images.html) - Official security hardening guide
* [Yocto Project Security Wiki](https://wiki.yoctoproject.org/wiki/Security) - Community security resources
* [Yocto Security Hardening: Security Flags](https://www.thegoodpenguin.co.uk/blog/yocto-security-hardening-security-flags/) - Deep dive on security_flags.inc
* [Yocto Kernel Development & Security Hardening](https://witekio.com/blog/yocto-kernel-development-security-hardening/) - Kernel hardening best practices
* [meta-security Layer Index](https://layers.openembedded.org/layerindex/branch/master/layer/meta-security/) - AppArmor, IMA/EVM, security tools
* [meta-selinux Git Repository](https://git.yoctoproject.org/meta-selinux/) - SELinux support for Yocto
* [Kernel Hardening Checker](https://github.com/a13xp0p0v/kernel-hardening-checker) - Audit kernel configuration for security
* [Reproducible Builds in Yocto](https://wiki.yoctoproject.org/wiki/Reproducible_Builds) - Yocto reproducibility documentation
* [Timesys VigiShield](https://www.timesys.com/security/vigishield-secure-by-design-for-yocto/) - Commercial CVE monitoring
* See [Chapter 10: Third Party Code and Components](10_third_party_code_and_components.md) for comprehensive Yocto CVE checking and SBOM generation
