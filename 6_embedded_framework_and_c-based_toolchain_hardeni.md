# Embedded Framework and C-Based Toolchain Hardening

Limit BusyBox, embedded frameworks, and toolchains to only those libraries and functions being used when configuring firmware builds. Embedded Linux build systems such as Buildroot, Yocto and others typically perform this task. Removal of known insecure libraries and protocols such as Telnet not only minimize attack entry points in firmware builds, but also provide a secure-by-design approach to building software in efforts to thwart potential security threats.

**Hardening a library** [**Example**](https://cheatsheetseries.owasp.org/cheatsheets/C-Based_Toolchain_Hardening_Cheat_Sheet.html)**:** It is known that [compression is insecure](http://arstechnica.com/security/2012/09/crime-hijacks-https-sessions/) (amongst others),[SSLv2 is insecure](http://www.schneier.com/paper-ssl-revised.pdf), [SSLv3 is insecure](http://www.yaksman.org/\~lweith/ssl.pdf), as well as early versions of TLS . In addition, suppose you don't use hardware and engines, and only allow static linking. Given the knowledge and specifications, you would configure the OpenSSL library as follows:

```bash
$ Configure darwin64-x86_64-cc -no-hw -no-engine -no-comp -no-shared -no-dso -no-ssl2 -no-ssl3 --openssldir=
```

**Selecting one shell Example**: Utilizing buildroot, the screenshot below demonstrates only one Shell being enabled, bash. (_Note: Buildroot examples are shown below but there are other ways to accomplish the same configuration with other embedded Linux build systems._)\


![](.gitbook/assets/embedSec3.png)

**Hardening Services Example**: The screenshot below shows openssh enabled but not FTP daemons proftpd and pure-ftpd. Only enable FTP if TLS is to be utilized. For example, proftpd and pureftpd require custom compilation to use TLS with mod\_tls for proftpd and passing `./configure --with-tls` for pureftpd.

![](.gitbook/assets/embedSec2.png)

**Hardening Das U-boot Example:** Often, physical access to an embedded device enables attack paths to modify bootloader configurations. Below, example best practice configurations for `uboot_config` are provided_. Note: The_ `uboot_config` _file is typically auto generated depending on the build environment and specific board._&#x20;

Configure "Verified Boot" (secure boot) for U-Boot 2013.07 versions and above. Verified Boot is not enabled by default and requires board support with the below configurations required at the minimum.

`CONFIG_ENABLE_VBOOT=y #Enables Verified Boot`

`CONFIG_FIT_SIGNATURE=y #Enables signature verification of FIT images.`

`CONFIG_RSA=y #Enables RSA algorithm used for FIT image verification`

`CONFIG_OF_SEPARATE=y #Enables separate build of u-Boot from the device tree.`

`CONFIG_FIT=y #Enables support for Flat Image Tree (FIT) uImage format.`

`CONFIG_OF_CONTROL=y #Enables Flattened Device Tree (FDT) configuration.`

`CONFIG_OF_LIBFDT=y`

`CONFIG_DEFAULT_DEVICE_TREE=y #Specifies the default Device Tree used for the run-time configuration of U-Boot.`

Afterwards, a series of steps are needed for configuring Verified Boot. An example overview of building [Verified Boot for a Beaglebone black board](https://github.com/siemens/u-boot/blob/master/doc/uImage.FIT/beaglebone\_vboot.txt) is:

1. Build U-Boot for the board, with the verified boot options enabled.
2. Obtain a suitable Linux kernel (preferably the latest)
3. Create a Image Tree Source file (ITS) file describing how you want the kernel to be packaged, compressed and signed.
4. Create an RSA key pair with RSA2048 and use SHA256 hashing algorithm for authentication (**store your private key in a safe place and not hardcoded into firmware**)
5. Sign the kernel
6. Put the **public key** into U-Boot's image
7. Put U-Boot and the kernel onto the board
8. Test the image and boot configurations

In addition to the above, make the applicable configurations valid to the context of your embedded device. Below are notable configurations that can be made.

`CONFIG_BOOTDELAY -2. #Prevents access to u-boot's console when auto boot is used`

`CONFIG_CMD_USB=n #Disables basic USB support and the usb command`

`CONFIG_USB_UHCI: defines the lowlevel part.`

`CONFIG_USB_KEYBOARD: enables the USB Keyboard`

`CONFIG_USB_STORAGE: enables the USB storage devices`

`CONFIG_USB_HOST_ETHER: enables USB ethernet adapter support`

Disabling serial console output in U-Boot via the following configuration macros:

`CONFIG_SILENT_CONSOLE`

`CONFIG_SYS_DEVICE_NULLDEV`

`CONFIG_SILENT_CONSOLE_UPDATE_ON_RELOC`

To enable immutable U-boot environment variables to prevent unauthorized changes (e.g. Modifying bootargs, updating verified boot public keys etc.) or side-loading of firmware, remove non-volatile memory settings such as the following:

`#define CONFIG_ENV_IS_IN_MMC`

`#define CONFIG_ENV_IS_IN_NAND`

`#define CONFIG_ENV_IS_IN_NVRAM`

`#define CONFIG_ENV_IS_IN_SPI_FLASH`

`#define CONFIG_ENV_IS_IN_REMOTE`

`#define CONFIG_ENV_IS_IN_EEPROM`

`#define CONFIG_ENV_IS_IN_FLASH`

`#define CONFIG_ENV_IS_IN_DATAFLASH`

`#define CONFIG_ENV_IS_IN_MMC`

`#define CONFIG_ENV_IS_IN_FAT`

`#define CONFIG_ENV_IS_IN_ONENAND`

`#define CONFIG_ENV_IS_IN_UBI`

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
*   Remove unused/unnecessary utilities such as:

    * sed, wget, curl, awk, cut, df, dmesg, echo, fdisk, grep, mkdir, mount (vfat), printf, tail, tee, test (directory), test (file), head, cat

    [Automotive Grade Linux (AGL) has developed an example table](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/07-system-hardening.html#removal-or-non-inclusion-of-utilities) of common utilities and their usage for debug or production environments (builds).

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

## Yocto Project Build System Security (2024-2025)

The Yocto Project provides a comprehensive, production-grade embedded Linux build system with extensive built-in security features. As of 2025, Yocto has become the de facto standard for secure embedded Linux development in automotive, industrial, medical, and consumer IoT markets. This section covers security hardening capabilities specific to Yocto builds.

### Yocto Scarthgap 5.0 LTS Security Foundation (2024)

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
