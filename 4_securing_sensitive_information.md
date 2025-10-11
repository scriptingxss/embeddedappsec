# Securing Sensitive Information

Do not hardcode secrets such as passwords, usernames, tokens, private keys or similar variants into firmware release images. This also includes the storage of sensitive data that is written to disk. If hardware security element \(SE\) or Trusted Execution Environment \(TEE\) is available, it is recommended to utilize such features for storing sensitive data. Otherwise, use of strong cryptography should be evaluated to protect the data.

If possible, all sensitive data in clear-text should be ephemeral by nature and reside in a volatile memory only.

**Noncompliant** [**Hardcoded Password**](https://www.owasp.org/index.php/Use_of_hard-coded_password) **Example:**

```c
int VerifyAdmin(char *password) {

  if (strcmp(password, "Mew!")) {
    printf("Incorrect Password!\n");
    return 0;
  }

  printf("Entering Diagnostic Mode\n");
  return 1;
}
```

**Noncompliant** [**Storing sensitive data to disk**](https://wiki.sei.cmu.edu/confluence/display/c/MEM06-C.+Ensure+that+sensitive+data+is+not+written+out+to+disk) **Example:**

In this noncompliant code example, sensitive information is supposedly stored in the dynamically allocated buffer, secret, which is processed and eventually cleared by a call to `memset_s()`. The memory page containing secret can be swapped out to disk. If the program crashes before the call to `memset_s()` completes, the information stored in secret may be stored in the core dump.

```c
char *secret;

secret = (char *)malloc(size+1);
if (!secret) {
  /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret, '\0', size+1);
free(secret);
secret = NULL;
```

To prevent the information from being written to a core dump, the size of core dumps that the program will generate should be set to 0 using `setrlimit()`:

```c
#include <sys/resource.h>
/* ... */
struct rlimit limit;
limit.rlim_cur = 0;
limit.rlim_max = 0;
if (setrlimit(RLIMIT_CORE, &limit) != 0) {
    /* Handle error */
}

char *secret;

secret = (char *)malloc(size+1);
if (!secret) {
  /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret, '\0', size+1);
free(secret);
secret = NULL;
```

Alternatively, the use of `mlock()` can be used to prevent paging by locking memory in place. This compliant solution not only disables the creation of core files but also ensures that the buffer is not swapped to hard disk:

```c
#include <sys/resource.h>
/* ... */
struct rlimit limit;
limit.rlim_cur = 0;
limit.rlim_max = 0;
if (setrlimit(RLIMIT_CORE, &limit) != 0) {
    /* Handle error */
}

long pagesize = sysconf(_SC_PAGESIZE);
if (pagesize == -1) {
  /* Handle error */
}

char *secret_buf;
char *secret;

secret_buf = (char *)malloc(size+1+pagesize);
if (!secret_buf) {
  /* Handle error */
}

/* mlock() may require that address be a multiple of PAGESIZE */
secret = (char *)((((intptr_t)secret_buf + pagesize - 1) / pagesize) * pagesize);

if (mlock(secret, size+1) != 0) {
    /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret_buf, '\0', size+1+pagesize);
if (munlock(secret, size+1) != 0) {
    /* Handle error */
}
secret = NULL;

free(secret_buf);
secret_buf = NULL;
```

**Storing Sensitive Data, Noncompliant** [**Example**](https://wiki.sei.cmu.edu/confluence/display/c/MEM03-C.+Clear+sensitive+information+stored+in+reusable+resources): In this example, sensitive information stored in the dynamically allocated memory referenced by secret is copied to the dynamically allocated buffer, `new_secret`, which is processed and eventually deallocated by a call to `free()`. Because the memory is not cleared, it may be reallocated to another section of the program where the information stored in `new_secret` may be unintentionally leaked.

```c
char *secret;

/* Initialize secret */

char *new_secret;
size_t size = strlen(secret);
if (size == SIZE_MAX) {
  /* Handle error */
}

new_secret = (char *)malloc(size+1);
if (!new_secret) {
  /* Handle error */
}
strcpy(new_secret, secret);

/* Process new_secret... */

free(new_secret);
new_secret = NULL;
```

**Storing Sensitive Data, Compliant Example**: To prevent information leakage, dynamic memory containing sensitive information should be sanitized before being freed. Sanitization is commonly accomplished by clearing the allocated space \(that is, filling the space with '\0' characters\).

```c
char *secret;

/* Initialize secret */

char *new_secret;
size_t size = strlen(secret);
if (size == SIZE_MAX) {
  /* Handle error */
}

/* Use calloc() to zero-out allocated space */
new_secret = (char *)calloc(size+1, sizeof(char));
if (!new_secret) {
  /* Handle error */
}
strcpy(new_secret, secret);

/* Process new_secret... */

/* Sanitize memory  */
memset_s(new_secret, '\0', size);
free(new_secret);
new_secret = NULL;
```

**Considerations:**

* Do not hardcode certificates across product lines.
* Do not hardcode passwords across product lines.
* Do not store secrets in an unprotected storage location or external storage including within an EEPROM or flash.
* **Leverage hardware security when available** (see Hardware-Based Security section below)

## Hardware-Based Security for Sensitive Data (2025 Best Practices)

Modern embedded devices should leverage hardware security features to protect sensitive information. Hardware-based security provides protection even when software is compromised.

### Trusted Execution Environment (TEE)

A TEE is a secure area within the main processor that ensures code and data loaded inside are protected with respect to confidentiality and integrity.

**ARM TrustZone** (most common for ARM Cortex-A and Cortex-M):
* Creates isolated "Secure World" and "Normal World" execution environments
* Store cryptographic keys, credentials, and sensitive data in Secure World
* Only minimal, security-critical code runs in Secure World
* **Use cases**: Secure boot, key storage, biometric authentication, DRM, payment processing
* **Resources**: [ARM TrustZone](https://www.arm.com/technologies/trustzone-for-cortex-m), [OP-TEE](https://www.op-tee.org/)

**Intel SGX (Software Guard Extensions)**:
* Creates encrypted memory regions called "enclaves"
* Protects sensitive data even from privileged software and physical attacks
* Suitable for x86-based embedded systems

**Other TEE Solutions**:
* **RISC-V**: Keystone, MultiZone, Sanctum
* **AMD SEV** (Secure Encrypted Virtualization)
* **Apple Secure Enclave**

### Hardware Security Module (HSM) and Secure Elements (SE)

Dedicated cryptographic processors designed to protect the cryptographic key lifecycle.

**Popular Secure Elements for IoT/Embedded**:
* **Microchip ATECC608A/B** - I2C secure element, ECDSA P-256, AES-128, SHA-256
* **NXP EdgeLock SE050** - Common Criteria EAL 6+ certified, supports RSA & ECC
* **Infineon OPTIGA Trust M** - I2C interface, RSA-1024/2048, ECC P-256/384
* **STMicroelectronics STSAFE** - Secure element for IoT devices
* **Maxim DS28E38** - DeepCover secure authenticator

**Trusted Platform Module (TPM)**:
* TPM 2.0 standard for platform integrity
* Protected key storage in TPM's shielded locations
* Supports: RSA 2048, ECC P-256, SHA-1/SHA-256, HMAC
* **Software**: tpm2-tss, tpm2-tools for Linux

### Selection Criteria

When choosing hardware security for embedded devices, evaluate:

1. **Security Certifications**:
   * Common Criteria EAL 4+ for moderate security
   * Common Criteria EAL 5+ or FIPS 140-2/140-3 Level 3+ for high security
   * EMVCo, PCI PTS for payment applications

2. **Cryptographic Capabilities**:
   * Symmetric: AES-128/256, HMAC-SHA256
   * Asymmetric: RSA 2048/4096, ECC P-256/P-384/P-521
   * True Random Number Generator (TRNG)
   * Hardware crypto acceleration

3. **Key Storage & Management**:
   * Number of key slots (typically 4-16)
   * Key import/export restrictions
   * Key usage policies and access controls
   * Key destruction and zeroization

4. **Physical Security**:
   * Tamper detection and response
   * Side-channel attack resistance (DPA, SPA, timing, EM)
   * Fault injection protection
   * Secure packaging options

5. **Integration Requirements**:
   * **Interface**: I2C, SPI, 1-Wire, USB
   * **Supply**: Voltage range, power consumption
   * **Environment**: Operating temperature range
   * **Software**: Library support (cryptoauthlib, mbedTLS, OpenSSL PKCS#11)
   * **Cost**: Unit cost vs. security benefit

### Best Practices for Hardware Security Integration

1. **Defense in Depth - Use Multiple Layers**:
   * TEE for secure code execution
   * SE/HSM for root-of-trust and key storage
   * Software crypto for bulk operations

2. **Minimize Exposure**:
   * Never extract sensitive keys from hardware
   * Perform crypto operations inside secure boundary
   * Use hardware for signing, encryption, attestation
   * Keep plaintext keys out of application memory

3. **Secure Provisioning**:
   * Inject unique keys during manufacturing in secure facility
   * Never share keys across devices
   * Implement secure boot chain rooted in hardware

4. **Key Hierarchy**:
   * Root keys stored in tamper-resistant hardware (never exposed)
   * Derive operational keys using KDF
   * Support key rotation without compromising root keys
   * Use key wrapping for secure key transport

5. **Attestation and Identity**:
   * Use device-unique keys for authentication
   * Implement remote attestation using hardware roots of trust
   * Support certificate-based device identity

### Implementation Example

```c
// Example: Using a secure element for ECDSA signing
#include <cryptoauthlib.h>

int sign_firmware_hash(uint8_t *hash, uint8_t *signature) {
    // Initialize connection to secure element (e.g., ATECC608)
    if (atcab_init(&cfg_ateccx08a_i2c_default) != ATCA_SUCCESS) {
        return -1;
    }

    // Sign hash using private key in slot 0
    // Private key NEVER leaves the secure element
    if (atcab_sign(0, hash, signature) != ATCA_SUCCESS) {
        atcab_release();
        return -1;
    }

    atcab_release();
    return 0;
}

// Example: Secure key storage with ARM TrustZone
// Secure World API (runs in TEE)
int tee_store_key(uint32_t key_id, uint8_t *key, size_t key_len) {
    // Validate key_id and key_len
    if (key_id >= MAX_KEYS || key_len > MAX_KEY_SIZE) {
        return TEE_ERROR_BAD_PARAMETERS;
    }

    // Store in secure storage (encrypted, integrity protected)
    return secure_storage_write(key_id, key, key_len);
}

// Normal World application
int encrypt_sensitive_data(uint8_t *plaintext, size_t len) {
    TEE_Session session;
    uint8_t ciphertext[MAX_DATA_SIZE];

    // Open session to Trusted Application
    if (TEEC_OpenSession(&context, &session, &uuid,
                         TEEC_LOGIN_PUBLIC, NULL, NULL, NULL) != TEEC_SUCCESS) {
        return -1;
    }

    // Invoke encryption in secure world (key never exposed)
    TEEC_InvokeCommand(&session, CMD_ENCRYPT, &operation, NULL);

    TEEC_CloseSession(&session);
    return 0;
}
```

### OWASP Alignment

* **OWASP ISVS**: V3.3 (Secure Storage), V3.2 (Cryptographic Functions)
* **OWASP ISTG**: ISTG-FW-CRYPT (Cryptography Testing)
* **OWASP IoTGoat**: Practice identifying hardcoded secrets in firmware

### TEE and Hardware Security Integration with Yocto Project

The Yocto Project provides comprehensive support for integrating TEE (Trusted Execution Environment) and hardware security modules into embedded Linux builds. This section covers how to enable and configure these security features in Yocto-based systems.

#### OP-TEE Integration (ARM TrustZone)

**OP-TEE** is the most widely used open-source TEE for ARM TrustZone. Yocto's **meta-security** layer provides full OP-TEE support.

**Enable OP-TEE in Yocto**:

```bash
# Clone meta-security layer
git clone https://git.yoctoproject.org/meta-security

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-security"
```

**Configuration (local.conf)**:
```bitbake
# Enable OP-TEE
DISTRO_FEATURES:append = " optee"

# Include OP-TEE OS and client libraries
IMAGE_INSTALL:append = " optee-os optee-client optee-test"

# Machine-specific OP-TEE platform
# Example for Raspberry Pi 4:
OPTEE_PLATFORM = "rpi4"

# Example for i.MX8:
# OPTEE_PLATFORM = "imx-mx8qxpmek"
```

**OP-TEE Components in Yocto**:
- **optee-os**: TEE OS running in ARM Secure World
- **optee-client**: Normal World client libraries (libteec)
- **optee-test**: Test suite for OP-TEE functionality
- **optee-examples**: Example Trusted Applications (TAs)

**Machine-specific Configuration**:
```bitbake
# In machine configuration (e.g., conf/machine/myboard.conf)
MACHINE_FEATURES:append = " optee"

# U-Boot with TrustZone support
PREFERRED_VERSION_u-boot = "2024.01"
UBOOT_CONFIG = "optee"

# ATF (ARM Trusted Firmware) integration
SPL_BINARY = "bl31.bin"
```

**Creating a Trusted Application (TA) Recipe**:
```bitbake
# recipes-security/my-ta/my-ta_1.0.bb
SUMMARY = "My Trusted Application for OP-TEE"
LICENSE = "BSD-2-Clause"

inherit optee-ta

SRC_URI = "file://my_ta.c \
           file://my_ta.h \
           file://Makefile"

do_install() {
    install -d ${D}${nonarch_base_libdir}/optee_armtz
    install -m 0444 ${B}/*.ta ${D}${nonarch_base_libdir}/optee_armtz/
}

FILES:${PN} = "${nonarch_base_libdir}/optee_armtz/"
```

**Example TA Code** (my_ta.c):
```c
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>

#define CMD_ENCRYPT_DATA 0

// Encrypt sensitive data in secure world
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
                                       uint32_t cmd_id,
                                       uint32_t param_types,
                                       TEE_Param params[4]) {
    switch (cmd_id) {
    case CMD_ENCRYPT_DATA:
        // Get plaintext from Normal World
        uint8_t *plaintext = params[0].memref.buffer;
        size_t plaintext_len = params[0].memref.size;

        // Encrypt using key stored in secure storage
        // Key never leaves Secure World
        TEE_ObjectHandle key;
        TEE_OpenPersistentObject(TEE_STORAGE_PRIVATE,
                                  "my_aes_key", 16,
                                  TEE_DATA_FLAG_ACCESS_READ,
                                  &key);

        // Perform AES encryption in secure world
        // ... encryption code ...

        return TEE_SUCCESS;

    default:
        return TEE_ERROR_BAD_PARAMETERS;
    }
}
```

#### TPM 2.0 Support in Yocto

**Trusted Platform Module (TPM) 2.0** provides hardware-based root of trust for key storage and platform integrity.

**Enable TPM 2.0** (via meta-security):
```bitbake
# In local.conf
DISTRO_FEATURES:append = " tpm2"

# Include TPM 2.0 tools and libraries
IMAGE_INSTALL:append = " tpm2-tss tpm2-tools tpm2-abrmd"

# Optional: PKCS#11 interface for TPM
IMAGE_INSTALL:append = " tpm2-pkcs11"
```

**Kernel Configuration for TPM**:
```cfg
# Enable TPM support in kernel
CONFIG_TCG_TPM=y
CONFIG_TCG_TIS_CORE=y
CONFIG_TCG_TIS=y          # For LPC/SPI TPM
CONFIG_TCG_TIS_I2C=y      # For I2C TPM
CONFIG_TCG_CRB=y          # For Command Response Buffer interface

# TPM 2.0 specific
CONFIG_TCG_TPM2_HMAC=y
CONFIG_SECURITYFS=y
```

**TPM Integration Recipe Example**:
```bitbake
# recipes-security/tpm-app/tpm-app_1.0.bb
SUMMARY = "Application using TPM 2.0 for key storage"
LICENSE = "MIT"

DEPENDS = "tpm2-tss"

SRC_URI = "file://tpm_app.c"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -ltss2-esys -ltss2-mu \
          -o tpm_app ${WORKDIR}/tpm_app.c
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/tpm_app ${D}${bindir}
}
```

**Example TPM Usage Code**:
```c
#include <tss2/tss2_esys.h>

int store_key_in_tpm(uint8_t *key, size_t key_len) {
    ESYS_CONTEXT *esys_ctx;
    TPM2B_SENSITIVE_CREATE in_sensitive;
    TPM2B_PUBLIC in_public;
    TPM2_HANDLE object_handle;

    // Initialize ESAPI context
    Esys_Initialize(&esys_ctx, NULL, NULL);

    // Set up key attributes
    in_sensitive.sensitive.data.size = key_len;
    memcpy(in_sensitive.sensitive.data.buffer, key, key_len);

    in_public.publicArea.type = TPM2_ALG_KEYEDHASH;
    in_public.publicArea.objectAttributes =
        TPMA_OBJECT_FIXEDTPM |
        TPMA_OBJECT_FIXEDPARENT |
        TPMA_OBJECT_USERWITHAUTH;

    // Create object in TPM (key stored in TPM NVRAM)
    Esys_Create(esys_ctx, ESYS_TR_RH_OWNER,
                ESYS_TR_PASSWORD, ESYS_TR_NONE, ESYS_TR_NONE,
                &in_sensitive, &in_public,
                NULL, NULL, NULL, NULL, NULL, NULL);

    Esys_Finalize(&esys_ctx);
    return 0;
}
```

#### Secure Element Integration (I2C/SPI)

**Common Secure Elements** supported in Yocto:
- **Microchip ATECC608**: I2C secure crypto authenticator
- **NXP EdgeLock SE050**: Advanced secure element
- **Infineon OPTIGA Trust M**: IoT security controller

**Example: ATECC608 Integration**:

**Device Tree Configuration** (for I2C secure element):
```dts
// In your device tree overlay
&i2c1 {
    status = "okay";

    atecc608a@60 {
        compatible = "microchip,atecc608a";
        reg = <0x60>;
        status = "okay";
    };
};
```

**Yocto Recipe for CryptoAuthLib**:
```bitbake
# recipes-security/cryptoauthlib/cryptoauthlib_3.7.3.bb
SUMMARY = "Microchip CryptoAuthentication Library"
LICENSE = "MIT"

SRC_URI = "git://github.com/MicrochipTech/cryptoauthlib.git;protocol=https;branch=main"
SRCREV = "${AUTOREV}"

inherit cmake

EXTRA_OECMAKE = "-DATCA_HAL_I2C=ON"

do_install:append() {
    install -d ${D}${includedir}/cryptoauthlib
    install -m 0644 ${S}/lib/*.h ${D}${includedir}/cryptoauthlib/
}

FILES:${PN} += "${libdir}/*.so*"
```

**Application Recipe Using ATECC608**:
```bitbake
# recipes-apps/secure-app/secure-app_1.0.bb
SUMMARY = "Application using ATECC608 secure element"
LICENSE = "MIT"

DEPENDS = "cryptoauthlib"

SRC_URI = "file://secure_app.c"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -lcryptoauth \
          -o secure_app ${WORKDIR}/secure_app.c
}
```

#### Hardware Security Best Practices in Yocto

**1. Secure Boot Chain with Hardware Root of Trust**:
```bitbake
# In local.conf
# U-Boot verified boot with hardware-backed keys
UBOOT_SIGN_ENABLE = "1"
UBOOT_SIGN_KEYDIR = "${DEPLOY_DIR_IMAGE}/keys"
UBOOT_SIGN_KEYNAME = "dev"

# Use TPM or secure element for key storage
UBOOT_SIGN_KEYFILE = "tpm:object:0x81000001"
```

**2. Secure Storage with dm-crypt and TPM**:
```bitbake
# Enable dm-crypt for encrypted partitions
IMAGE_INSTALL:append = " cryptsetup"

# Seal encryption keys to TPM
IMAGE_INSTALL:append = " clevis clevis-tpm2"
```

**3. Read-Only Rootfs with Secure Elements**:
```bitbake
# Read-only root filesystem (prevents persistent malware)
IMAGE_FEATURES += "read-only-rootfs"

# Store mutable configuration in secure element
# Application retrieves config from ATECC608/TPM at boot
```

**4. Development vs. Production Key Provisioning**:
```bitbake
# In distro config for production builds
# Separate key directories for dev/prod
UBOOT_SIGN_KEYDIR:class-target = "${DEPLOY_DIR_IMAGE}/production_keys"
UBOOT_SIGN_KEYDIR:class-native = "${DEPLOY_DIR_IMAGE}/dev_keys"

# Production builds: require signed images
EXTRA_IMAGE_FEATURES:remove = "debug-tweaks"
```

**5. Attestation and Remote Provisioning**:
```bitbake
# Include attestation services
IMAGE_INSTALL:append = " tpm2-totp"  # TPM-based TOTP

# For cloud attestation
IMAGE_INSTALL:append = " azure-iot-sdk-c"  # Azure IoT with TPM
IMAGE_INSTALL:append = " aws-iot-device-sdk-cpp"  # AWS IoT with secure elements
```

#### Verification and Testing

**Verify OP-TEE is Running**:
```bash
# On target device
# Check TEE supplicant is running
ps aux | grep tee-supplicant

# Run OP-TEE test suite
optee_example_hello_world

# Check secure world version
tee-supplicant -v
```

**Verify TPM Functionality**:
```bash
# Check TPM is detected
ls /dev/tpm*

# Read TPM capabilities
tpm2_getcap properties-fixed

# Create a key in TPM
tpm2_createprimary -C o -g sha256 -G rsa -c primary.ctx
tpm2_create -C primary.ctx -g sha256 -G rsa -r key.priv -u key.pub
```

**Verify Secure Element** (ATECC608):
```bash
# Use I2C tools to detect device
i2cdetect -y 1  # Should show device at 0x60

# Read device info using cryptoauthlib
./cryptoauth-test --device info
```

#### Integration with Other Chapters

For complete hardware security implementation in Yocto:
- **Secure Boot**: See [Chapter 3: Firmware Updates and Cryptographic Signatures](3_firmware_updates_and_cryptographic_signatures.md)
- **Kernel Hardening**: See [Chapter 6: Embedded Platform Security Hardening](6_embedded_framework_and_c-based_toolchain_hardeni.md) - Yocto kernel security section
- **Build System Security**: See [Chapter 6: Platform Hardening](6_embedded_framework_and_c-based_toolchain_hardeni.md) - Yocto Project Build System Security

#### Resources

**Yocto Project Documentation**:
- [Yocto meta-security Layer](https://layers.openembedded.org/layerindex/branch/master/layer/meta-security/)
- [OP-TEE in Yocto](https://optee.readthedocs.io/en/latest/building/devices/rpi3.html)

**Hardware Vendor Resources**:
- [Microchip ATECC608 Yocto Integration](https://github.com/MicrochipTech/cryptoauthlib/tree/main/lib)
- [NXP i.MX Security](https://www.nxp.com/design/software/embedded-software/i-mx-software/embedded-linux-for-i-mx-applications-processors:IMXLINUX) - includes EdgeLock SE050
- [Infineon OPTIGA Trust M](https://github.com/Infineon/optiga-trust-m)

**TPM Resources**:
- [TPM 2.0 Software Stack (TSS)](https://github.com/tpm2-software)
- [TPM 2.0 Tools Documentation](https://tpm2-tools.readthedocs.io/)

## Post-Quantum Cryptography for Secure Storage (2025+)

### Quantum Threat to Stored Data

**Retroactive Decryption Risk**:
- Encrypted data stored today can be decrypted with future quantum computers
- Embedded devices store: keys, credentials, configurations, user data
- Data lifetime often exceeds device lifetime (backups, forensics)

**Algorithms at Risk**:
- RSA encryption: Broken by Shor's algorithm
- ECC encryption (ECDH key exchange): Broken by Shor's algorithm
- Symmetric encryption (AES-256): Partially weakened (Grover's algorithm reduces to AES-128 effective security)

### NIST PQC Key Encapsulation Mechanisms (KEMs)

**ML-KEM (formerly Kyber)** - FIPS 203:
- Key encapsulation for encryption
- Three security levels:
  - ML-KEM-512: Equivalent to AES-128 (quantum-resistant)
  - ML-KEM-768: Equivalent to AES-192
  - ML-KEM-1024: Equivalent to AES-256
- Ciphertext sizes: 768-1568 bytes
- Public key sizes: 800-1568 bytes

### Hybrid Encryption Approach

**Recommended Strategy**:
```
Hybrid KEM = Classical ECDH + ML-KEM
- Generates shared secret using both algorithms
- Secure if either algorithm remains unbroken
```

**Implementation Pattern**:
```c
#include <openssl/evp.h>
#include "oqs/oqs.h"  // liboqs

// Hybrid key encapsulation
int hybrid_encrypt_data(const uint8_t *plaintext, size_t pt_len,
                        uint8_t **ciphertext, size_t *ct_len,
                        const uint8_t *ecdh_pubkey,
                        const uint8_t *mlkem_pubkey) {

    // 1. Classical ECDH key exchange
    uint8_t ecdh_secret[32];
    ecdh_derive_secret(ecdh_pubkey, ecdh_secret, 32);

    // 2. PQC ML-KEM encapsulation
    uint8_t mlkem_secret[32];
    uint8_t mlkem_ciphertext[1088];  // ML-KEM-768 ciphertext

    OQS_KEM *kem = OQS_KEM_new(OQS_KEM_alg_ml_kem_768);
    OQS_KEM_encaps(kem, mlkem_ciphertext, mlkem_secret, mlkem_pubkey);

    // 3. Combine secrets using KDF
    uint8_t combined_secret[32];
    HKDF_extract(ecdh_secret, 32, mlkem_secret, 32, combined_secret, 32);

    // 4. Encrypt data with AES-256-GCM using combined secret
    encrypt_aes_gcm(plaintext, pt_len, combined_secret, ciphertext, ct_len);

    // 5. Prepend ML-KEM ciphertext to encrypted data
    // Format: [ML-KEM ciphertext (1088)] [AES-GCM ciphertext + tag]

    OQS_KEM_free(kem);
    return 0;
}
```

### Symmetric Key Upgrade for Quantum Resistance

**AES Key Sizes**:
- AES-128: ~64-bit quantum security (Grover's algorithm)
- AES-192: ~96-bit quantum security
- **AES-256: ~128-bit quantum security** ← Use this

**Recommendation**: Upgrade all embedded storage encryption to **AES-256**

**Implementation**:
```c
// Quantum-resistant symmetric encryption
#include <openssl/evp.h>

int encrypt_storage_quantum_safe(const uint8_t *plaintext, size_t len,
                                  uint8_t *ciphertext,
                                  const uint8_t *key_256bit) {
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    int outlen;

    // Use AES-256-GCM (128-bit quantum security)
    EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, key_256bit, iv);
    EVP_EncryptUpdate(ctx, ciphertext, &outlen, plaintext, len);
    EVP_EncryptFinal_ex(ctx, ciphertext + outlen, &outlen);

    EVP_CIPHER_CTX_free(ctx);
    return 0;
}
```

### TEE/HSM Integration for PQC

**Hardware Support Status (2025)**:
- ARM TrustZone: Experimental PQC support via mbedTLS PQC branch
- TPM 2.0: No native PQC support (use software implementation in TEE)
- Secure Elements: Limited PQC support (check vendor roadmap)

**Hybrid Approach with TEE**:
```c
// Store PQC keys in TEE, perform crypto in secure world
int tee_hybrid_decrypt(uint32_t key_id,
                       const uint8_t *ciphertext, size_t ct_len,
                       uint8_t *plaintext, size_t *pt_len) {
    TEEC_Session session;
    TEEC_Operation op;

    // Invoke TEE Trusted Application for PQC decryption
    TEEC_InvokeCommand(&session, CMD_PQC_DECRYPT, &op, NULL);

    // PQC private keys never leave TEE
    return 0;
}
```

### Storage Overhead Planning

**Encrypted Data Size Increases**:
- ML-KEM-768 ciphertext: +1088 bytes per encryption
- ML-KEM-1024 ciphertext: +1568 bytes per encryption
- For bulk data: Hybrid KEM once, then AES-256

**Flash Planning**:
- Budget 1-2KB overhead per encrypted object
- Consider compression before encryption
- Use ML-KEM-512 for resource-constrained devices

### Migration Strategy

**Phase 1 (2025-2026)**: Parallel Encryption
- Encrypt new data with hybrid PQC + classical
- Keep existing data in classical encryption
- Add PQC support to TEE/SE firmware

**Phase 2 (2026-2028)**: Progressive Re-encryption
- Re-encrypt stored secrets during key rotation
- Prioritize long-lived data (certificates, keys)
- Monitor performance impact

**Phase 3 (2028+)**: PQC-Only Storage
- All new storage uses PQC algorithms
- Deprecate classical-only encryption
- Maintain hybrid for legacy compatibility

### OWASP IoT Ecosystem Integration

**OWASP ISVS Alignment**:
- V3.3.1: Cryptographic protection of sensitive data at rest
- V3.3.2: Secure key storage mechanisms
- V3.2.1: Use of approved cryptographic algorithms

**OWASP ISTG Testing**:
- ISTG-FW-CRYPT-001: Test cryptographic implementation strength
- ISTG-DES-LOGIC-001: Verify key storage security

**OWASP FSTM**:
- Stage 4: Dynamic analysis of encryption implementation
- Stage 6: Runtime testing of key extraction resistance

**OWASP IoTGoat**:
- Practice extracting hardcoded keys from firmware
- Test secure storage implementations

### Resources

- [NIST PQC Standards](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [FIPS 203 (ML-KEM)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf)
- [liboqs - Open Quantum Safe](https://github.com/open-quantum-safe/liboqs)
- [OQS OpenSSL Provider](https://github.com/open-quantum-safe/oqs-provider)

## Additional References <a id="additional-references"></a>

### CWE References
* [CWE-259: Use of Hard-coded Password](https://cwe.mitre.org/data/definitions/259.html)
* [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)
* [CWE-321: Use of Hard-coded Cryptographic Key](https://cwe.mitre.org/data/definitions/321.html)
* [CWE-321: CVE-2013-6952](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2013-6952) - Belkin hardcoded crypto key

### Hardware Security Resources
* [ARM TrustZone for Cortex-M](https://www.arm.com/technologies/trustzone-for-cortex-m)
* [OP-TEE (Open Portable TEE)](https://www.op-tee.org/) - Open-source TEE
* [GlobalPlatform TEE Specifications](https://globalplatform.org/specs-library/tee-specifications/)
* [TPM 2.0 Library Specification](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
* [Microchip Trust Platform](https://www.microchip.com/en-us/products/security/trust-platform) - Secure element solutions
* [NXP EdgeLock](https://www.nxp.com/products/security-and-authentication/authentication/edgelock-secure-authenticator:SE050) - Secure element family

### Standards and Certifications
* [Common Criteria](https://www.commoncriteriaportal.org/) - Security evaluation standard
* [FIPS 140-2/140-3](https://csrc.nist.gov/projects/cryptographic-module-validation-program) - Cryptographic module validation
* [EMVCo Security Evaluation](https://www.emvco.com/emv-technologies/security-evaluation/) - Payment security
* [PSA Certified](https://www.psacertified.org/) - IoT security certification (ARM)

