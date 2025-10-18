# Firmware Updates and Cryptographic Signatures

## Cryptographic Firmware Verification Principles

Modern embedded systems use cryptographic signatures to ensure firmware authenticity and integrity. This prevents attackers from installing malicious firmware, even with physical device access.

**Core Principles**:
1. **Asymmetric Cryptography**: Private key signs firmware (kept secure), public key verifies (embedded in device)
2. **Chain of Trust**: Each boot stage verifies the next (ROM → Bootloader → Kernel → Rootfs)
3. **Anti-Rollback**: Prevent downgrade attacks to vulnerable firmware versions
4. **Secure Storage**: Public keys stored in one-time-programmable (OTP) memory or secure elements

### Modern Firmware Signing: U-Boot FIT Images

**FIT (Flattened Image Tree)** is the modern standard for embedded Linux firmware images, replacing legacy uImage format.

**FIT Image Structure**:
```
firmware.itb (Flattened Image Tree Binary)
├── Kernel image (compressed)
├── Device tree blob (DTB)
├── Initramfs (optional)
└── Digital signatures (RSA-2048/4096 or ECDSA P-256/384)
```

**Example: Creating and Signing a FIT Image**

**1. Generate RSA Key Pair** (do this once, protect private key!):
```bash
# Generate 4096-bit RSA key pair
openssl genpkey -algorithm RSA -out keys/dev-private.key -pkeyopt rsa_keygen_bits:4096

# Extract public key
openssl rsa -in keys/dev-private.key -pubout -out keys/dev-public.key
```

**2. Create FIT Image Source (.its file)**:
```dts
/dts-v1/;

/ {
    description = "Signed firmware for production device";
    #address-cells = <1>;

    images {
        kernel {
            description = "Linux Kernel 6.6 LTS";
            data = /incbin/("./Image.gz");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            load = <0x80080000>;
            entry = <0x80080000>;
            hash-1 {
                algo = "sha256";
            };
        };

        fdt {
            description = "Device Tree Blob";
            data = /incbin/("./device-tree.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash-1 {
                algo = "sha256";
            };
        };
    };

    configurations {
        default = "config-1";
        config-1 {
            description = "Production Configuration";
            kernel = "kernel";
            fdt = "fdt";
            signature {
                algo = "sha256,rsa4096";
                key-name-hint = "dev";
                sign-images = "kernel", "fdt";
            };
        };
    };
};
```

**3. Build and Sign FIT Image**:
```bash
# Create unsigned FIT image
mkimage -f firmware.its firmware-unsigned.itb

# Sign with private key
mkimage -F -k keys/ -K u-boot.dtb -r firmware-unsigned.itb

# Result: firmware.itb (signed)
```

**4. Verify Signature** (U-Boot bootloader):
```bash
# U-Boot will automatically verify signature before booting
# If signature invalid, boot process halts

# Manual verification for testing:
fit_check firmware.itb
```

**Production Key Management**:
- **Development keys**: Used during development, keys kept in source control
- **Production keys**: Stored in Hardware Security Module (HSM), accessed only by CI/CD
- **Key rotation**: Plan for key compromise (include public key version in FIT)
- **Secure boot chain**: Public key hash burned into SoC OTP/eFuses

**Hardware Root of Trust Integration**:
- TPM 2.0: Store public keys in TPM NVRAM, verify with `tpm2_verifysignature`
- OP-TEE: Verify signatures in Trusted Execution Environment
- Secure Element: Offload signature verification to ATECC608, EdgeLock SE050

For comprehensive Yocto Project integration of U-Boot verified boot, see [Yocto Project Secure Boot Implementation](#yocto-project-secure-boot-implementation) below.

---

**Considerations:**

* Ensure robust update mechanisms utilize cryptographically signed firmware images for updating functions.
  * GPG \([https://github.com/romanz/trezor-agent/blob/master/README-GPG.md](https://github.com/romanz/trezor-agent/blob/master/README-GPG.md)\)
* Ensure updates are downloaded over the most recent secure TLS version possible. \(As of writing, this is TLS1.3\)
  * Ensure updates validate the public key and certificate chain of the update server.
* Include a feature to utilize automatic firmware updates upon a predefined schedule.
  * Force updates in highly vulnerable use cases.
  * Scheduled push updates should be taken into consideration for certain devices, such as medical devices, to prevent force updates from creating possible issues.
* Ensure firmware versions are clearly displayed.
* Ensure firmware updates include changelogs with security related vulnerabilities included.
* Ensure an anti downgrade protection \(anti-rollback\) mechanism is employed so that the device cannot be reverted to a vulnerable version.
* Consider implementing an [Integrity Measurement Architecture \(IMA\)](https://sourceforge.net/p/linux-ima/wiki/Home/) which allows the kernel to check that a file has not been changed by validating it against a stored/calculate hash \(called label\) while Extended Verification Module \(EVM\) checks the file attributes \(including the extended ones\).
  * There are two types of labels are available :
    * immutable and signed
    * Simple
* Consider implementing a [read only root file system](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/05-security-concepts.html#read-only-root-file-system) with an overlay that can be created for directories which require local persistence.
  * Apply appropriate controls and monitoring for approved processes that can write data to persistent storage locations. 
* Ensure a reliable clock source is available for querying certificate revocation servers.

## Post-Quantum Cryptography Readiness for Secure Boot

### The Quantum Threat to Embedded Systems

**Harvest Now, Decrypt Later (HNDL) Attacks**:
- Adversaries collect encrypted firmware/updates today
- Decrypt when quantum computers become available (10-15 years)
- Embedded devices have 10-20+ year lifecycles - vulnerable!

**Impact on Secure Boot**:
- Current RSA-2048/4096 signatures: Broken by Shor's algorithm on quantum computers
- Current ECDSA P-256/384 signatures: Broken by Shor's algorithm
- Firmware authenticity verification becomes useless

### NIST PQC Standards (Finalized 2024)

**Digital Signatures** (for secure boot):

1. **ML-DSA (formerly Dilithium)** - FIPS 204
   - Signature sizes: 2420-4627 bytes (vs RSA-2048: 256 bytes)
   - Public key: 1312-2592 bytes
   - Best for: General-purpose embedded systems

2. **SLH-DSA (formerly SPHINCS+)** - FIPS 205
   - Signature sizes: 7856-49856 bytes (very large!)
   - Stateless hash-based signatures
   - Best for: High-security, code-signing when size permits

3. **FN-DSA (Falcon)** - Round 4 candidate
   - Signature sizes: 666-1280 bytes (compact)
   - Best for: Resource-constrained embedded devices

### Hybrid Cryptography Approach (Recommended)

**Transition Strategy**:
```
Secure Boot Signature = Classical Signature || PQC Signature

Example:
- ECDSA P-384 (104 bytes) + ML-DSA-65 (3309 bytes) = ~3.4KB total
- Provides security if either algorithm remains secure
```

**Implementation Roadmap**:

**Phase 1 (2025-2026): Preparation**
- Audit current signature sizes and bootloader limitations
- Evaluate PQC library options (liboqs, PQClean, Kyber)
- Test hybrid signature verification performance
- Update bootloader to support larger signatures

**Phase 2 (2026-2027): Hybrid Deployment**
- Deploy hybrid classical+PQC signatures
- Maintain backward compatibility with classical-only validation
- Monitor performance impact on boot times

**Phase 3 (2028+): PQC-Only**
- Transition to PQC-only signatures
- Deprecate classical-only algorithms

### Bootloader PQC Implementation Example

```c
// Hybrid signature verification (ECDSA + ML-DSA)
#include <openssl/evp.h>
#include "pqcrypto.h"  // NIST PQC library

#define ECDSA_SIG_SIZE 104
#define MLDSA_SIG_SIZE 3309
#define HYBRID_SIG_SIZE (ECDSA_SIG_SIZE + MLDSA_SIG_SIZE)

typedef struct {
    uint8_t ecdsa_sig[ECDSA_SIG_SIZE];
    uint8_t mldsa_sig[MLDSA_SIG_SIZE];
} hybrid_signature_t;

int verify_firmware_hybrid(const uint8_t *firmware, size_t fw_len,
                           const hybrid_signature_t *sig,
                           const uint8_t *ecdsa_pubkey,
                           const uint8_t *mldsa_pubkey) {
    int result = 0;

    // Verify ECDSA signature (classical)
    EVP_PKEY *ec_key = /* load ECDSA public key */;
    result = ecdsa_verify(firmware, fw_len, sig->ecdsa_sig, ec_key);
    if (result != 1) {
        printf("ECDSA verification failed\n");
        return 0;
    }

    // Verify ML-DSA signature (post-quantum)
    result = mldsa_verify(firmware, fw_len, sig->mldsa_sig,
                          mldsa_pubkey, MLDSA_65);
    if (result != 1) {
        printf("ML-DSA verification failed\n");
        return 0;
    }

    // Both signatures valid - firmware authenticated
    printf("Hybrid signature verification: SUCCESS\n");
    return 1;
}
```

### Hardware Acceleration for PQC

**Requirements**:
- PQC algorithms are computationally intensive
- Consider hardware crypto accelerators
- ARM CryptoCell, Intel QAT, or custom FPGA acceleration

**Performance Targets**:
- Boot time impact: < 500ms additional for PQC verification
- Memory overhead: 10-20KB for PQC libraries
- Flash overhead: 3-5KB per signature

### Storage Considerations

**Signature Storage**:
- Classical RSA-2048: 256 bytes
- Hybrid (ECDSA + ML-DSA): ~3.4KB
- **Impact**: 13x increase - plan flash partitions accordingly

**Public Key Storage**:
- Store PQC public keys in secure boot ROM or OTP
- Use key derivation if multiple keys needed
- Consider compression for large ML-DSA keys

### OWASP IoT Ecosystem Integration

**OWASP ISVS Alignment**:
- V3.2.1: Use of cryptographically signed firmware updates
- V3.2.2: Verification of digital signatures before installation
- V3.2.3: Protection against firmware downgrade attacks

**OWASP ISTG Testing**:
- ISTG-FW-INFO-001: Verify firmware signature validation
- ISTG-FW-CRYPT-001: Test cryptographic signature strength

**OWASP FSTM**:
- Stage 3: Extracting firmware and analyzing signature implementation
- Stage 5: Runtime analysis of secure boot process

**OWASP IoTGoat**:
- Practice extracting and analyzing firmware signatures
- Test downgrade attack prevention mechanisms

### Resources

- [NIST PQC Project](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [liboqs - Open Quantum Safe](https://github.com/open-quantum-safe/liboqs)
- [FIPS 204 (ML-DSA)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf)
- [PQC for Embedded - BSI](https://www.bsi.bund.de/EN/Themen/Unternehmen-und-Organisationen/Informationen-und-Empfehlungen/Kryptografie/Postquantenkryptografie/postquantenkryptografie_node.html)

## Yocto Project Secure Boot Implementation

The Yocto Project provides comprehensive support for implementing secure boot chains using U-Boot Verified Boot (FIT image signing). This ensures that only cryptographically signed firmware images can execute on the device.

### U-Boot Verified Boot Configuration

**Enable Verified Boot in Yocto**:

```bitbake
# In local.conf
# Enable U-Boot FIT image signing
UBOOT_SIGN_ENABLE = "1"

# Specify key directory (development keys)
UBOOT_SIGN_KEYDIR = "${DEPLOY_DIR_IMAGE}/keys"
UBOOT_SIGN_KEYNAME = "dev"

# Device tree options for signing
UBOOT_MKIMAGE_DTCOPTS = "-I dts -O dtb -p 2000"

# Sign kernel FIT image
KERNEL_IMAGETYPE = "fitImage"
KERNEL_CLASSES += "kernel-fitimage"
KERNEL_SIGN_ENABLE = "1"
```

**Generate Signing Keys**:

```bash
# Create key directory
mkdir -p build/tmp/deploy/images/keys

# Generate RSA 4096-bit key for signing
openssl genpkey -algorithm RSA -out build/tmp/deploy/images/keys/dev.key \
        -pkeyopt rsa_keygen_bits:4096

# Generate certificate
openssl req -batch -new -x509 -key build/tmp/deploy/images/keys/dev.key \
        -out build/tmp/deploy/images/keys/dev.crt
```

**Build Signed Image**:

```bash
# Build kernel with FIT image signing
bitbake virtual/kernel

# Build complete image with signed bootloader and kernel
bitbake core-image-minimal
```

### Kernel FIT Image Signing

**Kernel Recipe Configuration**:

```bitbake
# In linux-yocto_%.bbappend or local.conf
KERNEL_IMAGETYPE = "fitImage"
KERNEL_CLASSES = "kernel-fitimage"
KERNEL_SIGN_ENABLE = "1"

# ITS (Image Tree Source) file for FIT image
KERNEL_DEVICETREE = "myboard.dtb"

# Sign configurations
UBOOT_SIGN_KEYNAME = "dev"
UBOOT_SIGN_KEYDIR = "${DEPLOY_DIR_IMAGE}/keys"
```

**Verification at Boot**:

U-Boot will verify the digital signature of the kernel FIT image before loading it. If verification fails, boot is aborted.

```
## Checking hash(es) for FIT Image at 82000000 ...
   Hash(es) for Image 0 (kernel): sha256+ OK
   Sign for Image 0 (kernel): rsa4096+ OK
   Hash(es) for Image 1 (fdt): sha256+ OK
```

### Hardware Root of Trust Integration

**TPM-backed Secure Boot**:

```bitbake
# Use TPM for key storage
DISTRO_FEATURES:append = " tpm2"
IMAGE_INSTALL:append = " tpm2-tss tpm2-tools"

# Reference TPM-stored key for signing
UBOOT_SIGN_KEYFILE = "tpm:object:0x81000001"
```

**Secure Element Integration**:

```bitbake
# For OP-TEE with secure storage
DISTRO_FEATURES:append = " optee"
IMAGE_INSTALL:append = " optee-os optee-client"

# Keys stored in OP-TEE secure storage
UBOOT_SIGN_KEYFILE = "optee:secure_storage:signing_key"
```

### Production vs. Development Keys

**Separate Key Management**:

```bitbake
# In production distro .conf
UBOOT_SIGN_KEYDIR:class-target = "${TOPDIR}/../production_keys"
UBOOT_SIGN_KEYDIR:class-native = "${TOPDIR}/development_keys"

# Lock bootloader in production
EXTRA_IMAGE_FEATURES:remove = "debug-tweaks"
UBOOT_CONFIG = "secure"
```

### Complete Secure Boot Chain

**Full Chain Configuration**:

```bitbake
# 1. ROM -> U-Boot SPL (signed by SoC vendor)
# 2. U-Boot SPL -> U-Boot (verified)
UBOOT_SIGN_ENABLE = "1"

# 3. U-Boot -> Kernel (FIT image signed)
KERNEL_SIGN_ENABLE = "1"

# 4. Kernel -> Rootfs (dm-verity)
IMAGE_FEATURES += "dm-verity"

# 5. Runtime integrity (IMA/EVM)
DISTRO_FEATURES:append = " ima"
IMAGE_INSTALL:append = " ima-evm-utils"
```

### Testing Secure Boot

**Verify Signature in FIT Image**:

```bash
# Extract FIT image
bitbake -c compile virtual/kernel

# View FIT image contents
fitdump tmp/deploy/images/myboard/fitImage

# Verify signature manually
fit_check_sign -f tmp/deploy/images/myboard/fitImage \
               -k tmp/deploy/images/keys/dev.crt
```

**Test Boot with Signed Image**:

```bash
# Boot device and observe U-Boot verification
# Should see: "Sign for Image 0 (kernel): rsa4096+ OK"

# Test with tampered image (should fail to boot)
# Modify fitImage and observe boot failure
```

### Integration with Chapters

For complete Yocto secure boot implementation:
- **Hardware Security**: See [Chapter 4: Securing Sensitive Information](4_securing_sensitive_information.md) - TEE and Hardware Security with Yocto
- **Kernel Hardening**: See [Chapter 6: Embedded Platform Security Hardening](6_embedded_framework_and_c-based_toolchain_hardeni.md) - Yocto kernel security
- **CVE Management**: See [Chapter 10: Third Party Components](10_third_party_code_and_components.md) - Yocto CVE checking

### Resources

- [Yocto Secure Boot Documentation](https://docs.yoctoproject.org/dev/dev-manual/securing-images.html#using-verified-boot)
- [U-Boot Verified Boot](https://source.denx.de/u-boot/u-boot/-/blob/master/doc/uImage.FIT/verified-boot.txt)
- [Automotive Grade Linux Secure Boot](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/07-system-hardening.html#hardened-boot)

## Modern OTA Update Frameworks for Embedded Linux

Over-the-Air (OTA) updates are critical for embedded devices deployed in the field. Modern OTA frameworks provide atomic updates, rollback capabilities, and robust error recovery. This section covers the three major open-source OTA frameworks for embedded Linux.

### SWUpdate - Software Update for Embedded Systems

**SWUpdate** is a flexible framework that supports multiple update strategies with built-in security features.

####  SWUpdate Architecture

**Key Components**:
- **Update Engine**: Handles update logic and verification
- **Handlers**: Support for various image types (raw, ubifs, tar, scripts)
- **Hawkbit Integration**: Cloud-based update management
- **Local Updates**: USB, SD card, or network-based

**Security Features**:
- Cryptographic signature verification (RSA, ECDSA)
- Encrypted updates (AES-256-CBC)
- Hash verification (SHA-256)
- Secure boot integration
- Anti-rollback protection

#### SWUpdate Yocto Integration

**Add SWUpdate to your Yocto build**:

```bitbake
# local.conf
IMAGE_INSTALL:append = " swupdate swupdate-www"

# Enable signing
INHERIT += "swupdate"

# Define signing keys
SWUPDATE_SIGNING = "RSA"
SWUPDATE_PRIVATE_KEY = "${TOPDIR}/swupdate-private.pem"

# Anti-rollback configuration
SWUPDATE_HW_COMPATIBILITY = "1.0"
```

**Generate SWUpdate image recipe**:

```bitbake
# recipes-support/swupdate/swupdate-image.bb
DESCRIPTION = "SWUpdate compound image"
LICENSE = "MIT"

inherit swupdate

SWUPDATE_IMAGES = "core-image-minimal"

# Define software components
SRC_URI = " \
    file://sw-description \
    file://emmcsetup.lua \
"

# Image artifacts
SWUPDATE_IMAGES_FSTYPES[core-image-minimal] = ".ext4.gz"
```

**sw-description File Example**:

```plaintext
software =
{
    version = "1.0.1";

    hardware-compatibility: [ "1.0" ];

    /* Firmware images */
    images: (
        {
            filename = "core-image-minimal.ext4.gz";
            device = "/dev/mmcblk0p2";
            type = "raw";
            compressed = "zlib";
            sha256 = "@core-image-minimal.ext4.gz";

            installed-directly = true;
        }
    );

    /* Boot environment update */
    bootenv: (
        {
            name = "bootpart";
            value = "2";
        },
        {
            name = "bootcount";
            value = "0";
        }
    );

    /* Pre/post update scripts */
    scripts: (
        {
            filename = "emmcsetup.lua";
            type = "lua";
            sha256 = "@emmcsetup.lua";
        }
    );
}
```

**Signing SWUpdate Images**:

```bash
# Generate RSA key pair (one-time setup)
openssl genrsa -out swupdate-private.pem 2048
openssl rsa -in swupdate-private.pem -out swupdate-public.pem -outform PEM -pubout

# Sign sw-description
openssl dgst -sha256 -sign swupdate-private.pem sw-description > sw-description.sig

# Create SWU package
FILES="sw-description sw-description.sig core-image-minimal.ext4.gz emmcsetup.lua"
for i in $FILES; do echo $i; done | cpio -ov -H crc > firmware-v1.0.1.swu
```

**SWUpdate Client Integration**:

```c
// swupdate-client.c - Programmatic update trigger
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

#define SOCKET_PATH "/tmp/swupdateprog"

int trigger_swupdate(const char *swu_image_path) {
    int sockfd;
    struct sockaddr_un addr;

    // Connect to SWUpdate progress socket
    sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket");
        return -1;
    }

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sockfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        perror("connect");
        close(sockfd);
        return -1;
    }

    // Send update command
    char command[512];
    snprintf(command, sizeof(command), "SOURCE %s", swu_image_path);
    write(sockfd, command, strlen(command));

    // Monitor progress
    char buffer[256];
    ssize_t n;
    while ((n = read(sockfd, buffer, sizeof(buffer) - 1)) > 0) {
        buffer[n] = '\0';
        printf("SWUpdate: %s\n", buffer);

        // Check for completion
        if (strstr(buffer, "SUCCESS") != NULL) {
            printf("Update successful!\n");
            close(sockfd);
            return 0;
        }
        if (strstr(buffer, "FAILURE") != NULL) {
            printf("Update failed!\n");
            close(sockfd);
            return -1;
        }
    }

    close(sockfd);
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <firmware.swu>\n", argv[0]);
        return 1;
    }

    return trigger_swupdate(argv[1]);
}
```

### RAUC - Robust Auto-Update Controller

**RAUC** provides a comprehensive solution for managing updates with A/B partition schemes and atomic updates.

#### RAUC Architecture

**Core Concepts**:
- **Slots**: Update targets (rootfs A, rootfs B, boot, appfs)
- **Bundles**: Signed, compressed update packages (.raucb)
- **System Configuration**: Defines slots, boot logic, and update policy
- **D-Bus API**: Integration with system daemons

**A/B Partition Scheme**:
```
/dev/mmcblk0p1: boot (shared)
/dev/mmcblk0p2: rootfs.0 (active)
/dev/mmcblk0p3: rootfs.1 (inactive)
/dev/mmcblk0p4: appfs (persistent data)
```

#### RAUC Yocto Integration

**Add RAUC to Yocto build**:

```bitbake
# local.conf
DISTRO_FEATURES:append = " rauc"
IMAGE_INSTALL:append = " rauc"

# Enable bundle generation
INHERIT += "bundle"

# Signing configuration
RAUC_KEY_FILE = "${TOPDIR}/rauc.key.pem"
RAUC_CERT_FILE = "${TOPDIR}/rauc.cert.pem"
```

**RAUC System Configuration** (`/etc/rauc/system.conf`):

```ini
[system]
compatible=MyProduct-1.0
bootloader=uboot
mountprefix=/mnt/rauc

[keyring]
path=/etc/rauc/ca.cert.pem

[slot.rootfs.0]
device=/dev/mmcblk0p2
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/mmcblk0p3
type=ext4
bootname=B

[slot.appfs.0]
device=/dev/mmcblk0p4
type=ext4
parent=rootfs.0

[slot.appfs.1]
device=/dev/mmcblk0p4
type=ext4
parent=rootfs.1
```

**Create RAUC Bundle Recipe**:

```bitbake
# recipes-core/bundles/update-bundle.bb
DESCRIPTION = "RAUC update bundle"
LICENSE = "MIT"

inherit bundle

# Define bundle contents
RAUC_BUNDLE_COMPATIBLE = "MyProduct-1.0"
RAUC_BUNDLE_SLOTS = "rootfs"
RAUC_SLOT_rootfs = "core-image-minimal"
RAUC_SLOT_rootfs[fstype] = "ext4"

# Signing
RAUC_KEY_FILE = "${TOPDIR}/rauc.key.pem"
RAUC_CERT_FILE = "${TOPDIR}/rauc.cert.pem"
```

**RAUC Update Process**:

```bash
# Install update bundle
rauc install firmware-v1.0.2.raucb

# Check update status
rauc status

# Example output:
# Compatible: MyProduct-1.0
# Booted from: rootfs.0 (/dev/mmcblk0p2)
#
# Slot States:
#   rootfs.0: active, booted, good
#   rootfs.1: inactive, bundle.version=1.0.2
```

**RAUC D-Bus Integration (C)**:

```c
// rauc-dbus-client.c
#include <gio/gio.h>
#include <stdio.h>

void on_install_complete(GObject *source, GAsyncResult *res, gpointer user_data) {
    GError *error = NULL;
    GVariant *result = g_dbus_proxy_call_finish((GDBusProxy*)source, res, &error);

    if (error) {
        fprintf(stderr, "Installation failed: %s\n", error->message);
        g_error_free(error);
        return;
    }

    gint32 status;
    gchar *message;
    g_variant_get(result, "(is)", &status, &message);

    if (status == 0) {
        printf("Update successful: %s\n", message);
    } else {
        printf("Update failed: %s\n", message);
    }

    g_variant_unref(result);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <bundle.raucb>\n", argv[0]);
        return 1;
    }

    GError *error = NULL;
    GDBusProxy *proxy;

    // Connect to RAUC D-Bus service
    proxy = g_dbus_proxy_new_for_bus_sync(
        G_BUS_TYPE_SYSTEM,
        G_DBUS_PROXY_FLAGS_NONE,
        NULL,
        "de.pengutronix.rauc",
        "/",
        "de.pengutronix.rauc.Installer",
        NULL,
        &error
    );

    if (error) {
        fprintf(stderr, "Failed to connect to RAUC: %s\n", error->message);
        g_error_free(error);
        return 1;
    }

    // Trigger installation
    g_dbus_proxy_call(
        proxy,
        "Install",
        g_variant_new("(s)", argv[1]),
        G_DBUS_CALL_FLAGS_NONE,
        -1,
        NULL,
        on_install_complete,
        NULL
    );

    // Run event loop
    GMainLoop *loop = g_main_loop_new(NULL, FALSE);
    g_main_loop_run(loop);

    g_object_unref(proxy);
    return 0;
}
```

### Mender - End-to-End OTA Solution

**Mender** provides a complete commercial OTA solution with server infrastructure, client, and enterprise features.

#### Mender Architecture

**Components**:
- **Mender Client**: Device-side update agent
- **Mender Server**: Update management backend (open-source or hosted)
- **Mender Artifact**: Update package format
- **State Scripts**: Pre/post-update hooks

**Update Flow**:
1. Device polls Mender Server for updates
2. Server provides update artifact with signature
3. Client verifies signature and downloads
4. Client installs to inactive partition
5. Client reboots to new partition
6. Client commits update if successful (or rolls back)

#### Mender Yocto Integration

**Add Mender layer**:

```bash
# Clone Mender layers
git clone -b scarthgap https://github.com/mendersoftware/meta-mender.git

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-mender/meta-mender-core"
BBLAYERS += "/path/to/meta-mender/meta-mender-demo"  # For testing
```

**Configure Mender in local.conf**:

```bitbake
# Inherit Mender classes
INHERIT += "mender-full"

# Define A/B partition layout
MENDER_STORAGE_DEVICE = "/dev/mmcblk0"
MENDER_BOOT_PART = "${MENDER_STORAGE_DEVICE}p1"
MENDER_ROOTFS_PART_A = "${MENDER_STORAGE_DEVICE}p2"
MENDER_ROOTFS_PART_B = "${MENDER_STORAGE_DEVICE}p3"
MENDER_DATA_PART = "${MENDER_STORAGE_DEVICE}p4"

# Mender Server configuration
MENDER_SERVER_URL = "https://mender.example.com"
MENDER_TENANT_TOKEN = "YOUR_TENANT_TOKEN"

# Artifact configuration
MENDER_ARTIFACT_NAME = "release-v1.0.3"

# Enable delta updates (bandwidth optimization)
IMAGE_FEATURES:append = " mender-image-delta"
```

**Mender Client Configuration** (`/etc/mender/mender.conf`):

```json
{
  "ServerURL": "https://mender.example.com",
  "TenantToken": "YOUR_TENANT_TOKEN",
  "UpdatePollIntervalSeconds": 1800,
  "InventoryPollIntervalSeconds": 86400,
  "RetryPollIntervalSeconds": 300,
  "RootfsPartA": "/dev/mmcblk0p2",
  "RootfsPartB": "/dev/mmcblk0p3",
  "DeviceTypeFile": "/var/lib/mender/device_type"
}
```

**Create Mender Artifact**:

```bash
# Build artifact from rootfs
mender-artifact write rootfs-image \
  --artifact-name release-v1.0.3 \
  --device-type MyDevice-v1 \
  --file core-image-minimal-MyDevice.ext4 \
  --output-path firmware-v1.0.3.mender

# Sign artifact
mender-artifact sign firmware-v1.0.3.mender \
  --key private.key \
  --output firmware-v1.0.3-signed.mender

# Create delta update (bandwidth optimization)
mender-artifact write rootfs-image \
  --artifact-name release-v1.0.3 \
  --device-type MyDevice-v1 \
  --file core-image-minimal-MyDevice.ext4 \
  --depends rootfs-image.checksum:1.0.2-checksum \
  --output-path firmware-v1.0.2-to-v1.0.3-delta.mender
```

**Mender State Scripts** (Custom update logic):

```bash
#!/bin/bash
# /etc/mender/scripts/ArtifactInstall_Enter_00

# Example: Stop critical services before update
systemctl stop my-app.service

# Backup configuration
cp -r /etc/my-app /data/backup/

exit 0
```

```bash
#!/bin/bash
# /etc/mender/scripts/ArtifactCommit_Leave_00

# Example: Verify application after successful update
if ! systemctl is-active --quiet my-app.service; then
    echo "Application failed to start after update"
    exit 1
fi

# Remove backup
rm -rf /data/backup/

exit 0
```

**Mender API Integration**:

```python
#!/usr/bin/env python3
# mender-api-client.py - Deploy update via Mender API

import requests
import json

MENDER_SERVER = "https://mender.example.com"
API_TOKEN = "your-api-token"

def deploy_update(artifact_name, device_group):
    """Deploy Mender artifact to device group"""

    headers = {
        "Authorization": f"Bearer {API_TOKEN}",
        "Content-Type": "application/json"
    }

    # Create deployment
    payload = {
        "name": f"Deployment of {artifact_name}",
        "artifact_name": artifact_name,
        "devices": [device_group]
    }

    response = requests.post(
        f"{MENDER_SERVER}/api/management/v1/deployments/deployments",
        headers=headers,
        json=payload
    )

    if response.status_code == 201:
        deployment_id = response.json()["id"]
        print(f"Deployment created: {deployment_id}")
        return deployment_id
    else:
        print(f"Deployment failed: {response.text}")
        return None

def check_deployment_status(deployment_id):
    """Check status of deployment"""

    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    response = requests.get(
        f"{MENDER_SERVER}/api/management/v1/deployments/deployments/{deployment_id}",
        headers=headers
    )

    if response.status_code == 200:
        data = response.json()
        print(f"Status: {data['status']}")
        print(f"Success: {data['stats']['success']}")
        print(f"Failure: {data['stats']['failure']}")
        print(f"Pending: {data['stats']['pending']}")
        return data
    else:
        print(f"Failed to get status: {response.text}")
        return None

if __name__ == "__main__":
    deployment_id = deploy_update("release-v1.0.3", "production-devices")
    if deployment_id:
        check_deployment_status(deployment_id)
```

## Multi-Component OTA Updates

Modern embedded systems often have multiple updatable components beyond the main firmware.

### Wireless Module Firmware Updates

**Example: WiFi/Bluetooth Module OTA**:

```c
// wifi-module-update.c - Update wireless module firmware
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

#define WLAN_FW_PATH "/lib/firmware/wifi-module-fw.bin"
#define WLAN_FW_UPDATE_IOCTL 0x8001

int update_wifi_firmware(const char *fw_path) {
    int fd;

    // Open WiFi device
    fd = open("/dev/wlan0", O_RDWR);
    if (fd < 0) {
        perror("Failed to open WiFi device");
        return -1;
    }

    // Trigger firmware update via ioctl
    if (ioctl(fd, WLAN_FW_UPDATE_IOCTL, fw_path) < 0) {
        perror("WiFi firmware update failed");
        close(fd);
        return -1;
    }

    printf("WiFi firmware updated successfully\n");
    close(fd);

    // Reload WiFi driver
    system("modprobe -r brcmfmac && modprobe brcmfmac");

    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <firmware.bin>\n", argv[0]);
        return 1;
    }

    return update_wifi_firmware(argv[1]);
}
```

**Yocto Integration for WiFi Firmware**:

```bitbake
# recipes-connectivity/wifi-firmware/wifi-firmware_1.0.bb
DESCRIPTION = "WiFi module firmware update utility"
LICENSE = "MIT"

SRC_URI = "file://wifi-module-update.c \
           file://wifi-firmware-v2.bin"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o wifi-firmware-update wifi-module-update.c
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 wifi-firmware-update ${D}${bindir}/

    install -d ${D}${base_libdir}/firmware
    install -m 0644 wifi-firmware-v2.bin ${D}${base_libdir}/firmware/
}

FILES:${PN} += "${base_libdir}/firmware/wifi-firmware-v2.bin"
```

### Cellular Modem Firmware Updates

**Example: LTE Modem OTA Update**:

```python
#!/usr/bin/env python3
# cellular-modem-update.py - Update LTE modem firmware

import serial
import time
import hashlib

class ModemUpdater:
    def __init__(self, port='/dev/ttyUSB2'):
        self.ser = serial.Serial(port, baudrate=115200, timeout=5)

    def send_at_command(self, cmd, expected_response='OK'):
        """Send AT command and wait for response"""
        self.ser.write(f"{cmd}\r\n".encode())
        time.sleep(0.5)

        response = self.ser.read(self.ser.in_waiting).decode()
        if expected_response in response:
            return True
        return False

    def enter_download_mode(self):
        """Enter modem firmware download mode"""
        print("Entering download mode...")

        # AT command to enter download mode (modem-specific)
        if not self.send_at_command('AT+QFASTBOOT', 'OK'):
            print("Failed to enter download mode")
            return False

        time.sleep(2)
        print("Modem in download mode")
        return True

    def upload_firmware(self, firmware_path):
        """Upload firmware to modem"""
        print(f"Uploading firmware: {firmware_path}")

        with open(firmware_path, 'rb') as f:
            firmware_data = f.read()

        # Calculate checksum
        checksum = hashlib.sha256(firmware_data).hexdigest()
        print(f"Firmware checksum: {checksum}")

        # Send firmware in chunks
        chunk_size = 4096
        total_chunks = len(firmware_data) // chunk_size + 1

        for i in range(total_chunks):
            chunk = firmware_data[i*chunk_size:(i+1)*chunk_size]
            self.ser.write(chunk)
            time.sleep(0.1)

            progress = (i + 1) / total_chunks * 100
            print(f"Progress: {progress:.1f}%", end='\r')

        print("\nFirmware upload complete")
        return True

    def verify_and_reboot(self):
        """Verify firmware and reboot modem"""
        print("Verifying firmware...")

        # Send verification command (modem-specific)
        if not self.send_at_command('AT+QFWVERIFY', 'OK'):
            print("Firmware verification failed")
            return False

        print("Firmware verified, rebooting modem...")
        self.send_at_command('AT+CFUN=1,1', 'OK')  # Reboot

        time.sleep(10)
        return True

    def check_firmware_version(self):
        """Query current firmware version"""
        self.ser.write(b'AT+CGMR\r\n')
        time.sleep(0.5)
        response = self.ser.read(self.ser.in_waiting).decode()
        print(f"Firmware version: {response}")

if __name__ == '__main__':
    import sys

    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <modem-firmware.bin>")
        sys.exit(1)

    updater = ModemUpdater()

    # Check current version
    updater.check_firmware_version()

    # Perform update
    if updater.enter_download_mode():
        if updater.upload_firmware(sys.argv[1]):
            if updater.verify_and_reboot():
                time.sleep(15)
                updater.check_firmware_version()
                print("Modem firmware update successful!")
            else:
                print("Modem firmware update failed during verification")
        else:
            print("Modem firmware upload failed")
    else:
        print("Failed to enter download mode")
```

### Power Supply Unit (PSU) Firmware Updates

**Example: Smart PSU OTA (via I2C)**:

```c
// psu-firmware-update.c - Update PSU firmware via I2C
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>

#define PSU_I2C_ADDR 0x58
#define PSU_FW_UPDATE_CMD 0xF0
#define PSU_FW_VERIFY_CMD 0xF1

int update_psu_firmware(const char *i2c_bus, const char *fw_path) {
    int fd;
    FILE *fw_file;
    unsigned char buffer[256];
    size_t bytes_read;

    // Open I2C bus
    fd = open(i2c_bus, O_RDWR);
    if (fd < 0) {
        perror("Failed to open I2C bus");
        return -1;
    }

    // Set I2C slave address
    if (ioctl(fd, I2C_SLAVE, PSU_I2C_ADDR) < 0) {
        perror("Failed to set I2C slave address");
        close(fd);
        return -1;
    }

    // Open firmware file
    fw_file = fopen(fw_path, "rb");
    if (!fw_file) {
        perror("Failed to open firmware file");
        close(fd);
        return -1;
    }

    printf("Starting PSU firmware update...\n");

    // Send firmware update command
    buffer[0] = PSU_FW_UPDATE_CMD;
    if (write(fd, buffer, 1) != 1) {
        perror("Failed to send update command");
        fclose(fw_file);
        close(fd);
        return -1;
    }

    usleep(100000);  // Wait 100ms for PSU to enter update mode

    // Send firmware data in chunks
    int chunk_num = 0;
    while ((bytes_read = fread(buffer, 1, sizeof(buffer), fw_file)) > 0) {
        if (write(fd, buffer, bytes_read) != bytes_read) {
            perror("Failed to write firmware chunk");
            fclose(fw_file);
            close(fd);
            return -1;
        }

        chunk_num++;
        printf("Sent chunk %d (%zu bytes)\r", chunk_num, bytes_read);
        fflush(stdout);

        usleep(50000);  // Wait 50ms between chunks
    }

    printf("\nFirmware upload complete, verifying...\n");

    // Send verification command
    buffer[0] = PSU_FW_VERIFY_CMD;
    if (write(fd, buffer, 1) != 1) {
        perror("Failed to send verify command");
        fclose(fw_file);
        close(fd);
        return -1;
    }

    // Read verification result
    usleep(500000);  // Wait 500ms for verification
    if (read(fd, buffer, 1) != 1) {
        perror("Failed to read verification result");
        fclose(fw_file);
        close(fd);
        return -1;
    }

    if (buffer[0] == 0x00) {
        printf("PSU firmware verification successful!\n");
    } else {
        printf("PSU firmware verification failed (code: 0x%02X)\n", buffer[0]);
        fclose(fw_file);
        close(fd);
        return -1;
    }

    fclose(fw_file);
    close(fd);

    printf("PSU firmware update complete\n");
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <i2c-bus> <firmware.bin>\n", argv[0]);
        fprintf(stderr, "Example: %s /dev/i2c-1 psu-firmware-v2.bin\n", argv[0]);
        return 1;
    }

    return update_psu_firmware(argv[1], argv[2]);
}
```

### Coordinated Multi-Component Update Strategy

**Example: Orchestrate updates across all components**:

```bash
#!/bin/bash
# multi-component-update.sh - Coordinate updates for all components

set -e

FIRMWARE_DIR="/tmp/firmware-update"
LOG_FILE="/var/log/multi-component-update.log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

verify_checksum() {
    local file=$1
    local expected_checksum=$2

    actual_checksum=$(sha256sum "$file" | awk '{print $1}')

    if [ "$actual_checksum" != "$expected_checksum" ]; then
        log "ERROR: Checksum mismatch for $file"
        return 1
    fi

    log "Checksum verified for $file"
    return 0
}

update_component() {
    local component=$1
    local firmware_file=$2
    local checksum=$3

    log "Updating $component..."

    # Verify checksum
    if ! verify_checksum "$firmware_file" "$checksum"; then
        return 1
    fi

    case $component in
        "main")
            rauc install "$firmware_file"
            ;;
        "wifi")
            wifi-firmware-update "$firmware_file"
            ;;
        "cellular")
            python3 /usr/bin/cellular-modem-update.py "$firmware_file"
            ;;
        "psu")
            psu-firmware-update /dev/i2c-1 "$firmware_file"
            ;;
        *)
            log "ERROR: Unknown component: $component"
            return 1
            ;;
    esac

    log "$component update complete"
    return 0
}

main() {
    log "=== Multi-Component Update Started ==="

    # Component update order (main firmware last for reboot)
    declare -A components
    components=(
        ["psu"]="$FIRMWARE_DIR/psu-fw-v2.bin:abc123def456..."
        ["wifi"]="$FIRMWARE_DIR/wifi-fw-v3.bin:789ghi012jkl..."
        ["cellular"]="$FIRMWARE_DIR/modem-fw-v4.bin:345mno678pqr..."
        ["main"]="$FIRMWARE_DIR/firmware-v1.0.4.raucb:901stu234vwx..."
    )

    # Update each component
    for component in psu wifi cellular main; do
        IFS=':' read -r firmware checksum <<< "${components[$component]}"

        if ! update_component "$component" "$firmware" "$checksum"; then
            log "ERROR: Failed to update $component, aborting"
            exit 1
        fi
    done

    log "=== All Components Updated Successfully ==="
    log "System will reboot in 10 seconds..."

    sleep 10
    reboot
}

main "$@"
```

## Additional References <a id="additional-references"></a>

* [https://www.kernel.org/signature.html](https://www.kernel.org/signature.html)
* [https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html)
* [http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r4.pdf](http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r4.pdf)
* [https://www.math.utah.edu/~beebe/PGP-notes.html](https://www.math.utah.edu/~beebe/PGP-notes.html)
* [CWE-321: Use of Hard-coded Cryptographic Key - CVE-2013-6952](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2013-6952)
  * [http://www.ioactive.com/pdfs/IOActive\_Belkin-advisory-lite.pdf](http://www.ioactive.com/pdfs/IOActive_Belkin-advisory-lite.pdf)
* [Implementing secure remote firmware updates](https://www.allegrosoft.com/wp-content/uploads/Secure-Firmware-Updates-Paper.pdf)
* [Code Integrity during execution by Automotive Grade Linux](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/05-security-concepts.html#code-integrity-during-execution)
* [Securing Software Updates for Automobiles](https://uptane.github.io/)

