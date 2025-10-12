# Identity Management

## Introduction

Modern identity management for embedded and IoT devices encompasses both **user identity** (authentication of people accessing devices) and **device identity** (authentication of devices themselves to networks and cloud services). This chapter addresses both aspects with emphasis on hardware-based device identity solutions that have become critical for Zero Trust architectures and secure IoT deployments in 2025.

User accounts within an embedded device should not be static in nature. Features that allow separation of user accounts for internal web management, internal console access, as well as remote web management and remote console access should be available to prevent automated malicious attacks.

**Considerations:**

* Static passwords utilized for web management and terminal access across product lines should not be used, or be removed as part of the release process.
  * If static default passwords NEED to be used in the interim, ensure that users are forced to change their passwords upon device setup and activation.
  * Ensure that users have the option to change all passwords/pins/passphrases etc. to ALL built in accounts.
* Pre-production account validation scripts should be adjusted in a similar fashion to those that validate WIFI and WPS passwords if applicable.
* Implement remote login and local login account features for users.
* Separation of users for SSH login and Admin login.
* Remote login should implement a temporary account lockout threshold to prevent automated brute force attacks.
* For web management interface:
  * Ensure Session IDs are sent in the request body, not in the URL.
  * Ensure Session IDs are invalidated when users log out.
  * Ensure active Session IDs are invalidated when passwords are changed.
  * If Session IDs are stored in a cookie, ensure that the cookie has the HttpOnly flag set.
  * Ensure Session IDs are random and change across sessions.
* Ensure usernames, passwords and cookies containing Session IDs are not sent over insecure protocols \(e.g. HTTP, FTP and Telnet\).
* Password complexity policies should be enforced to discourage easy to guess passwords such as “Password1”. A complex password should have the following attributes:
  * At least 10 characters or more in length
  * At least one upper-case letter
  * At least one numeric character
  * At least one lower-case letter
  * At least one special character
* Ensure EEPROMs are password protected enforcing complexity requirements.
* Employ key and certificate rotation policies

---

## Hardware-Based Device Identity

### Overview

As embedded devices increasingly participate in Zero Trust network architectures and cloud ecosystems, **hardware-based device identity** has become essential for secure authentication, attestation, and authorization. Unlike passwords or software-based credentials, hardware device identities are:

- **Immutable**: Burned into hardware during manufacturing
- **Unique**: Each device has cryptographically unique credentials
- **Unforgeable**: Private keys never leave secure hardware
- **Standards-based**: Leverage open standards (IEEE 802.1AR, TCG DICE, ARM PSA)

This section covers modern, **open-source** device identity solutions suitable for embedded Linux, RTOS, and bare-metal deployments.

---

### IEEE 802.1AR DevID (Device Identifier)

**Standard**: [IEEE 802.1AR-2018](https://standards.ieee.org/standard/802_1AR-2018.html)
**Purpose**: Secure device identity using X.509 certificates
**Use Cases**: Network Access Control (NAC), Zero Trust Network Access (ZTNA), device onboarding

#### How It Works

802.1AR defines two types of device identifiers:

1. **IDevID** (Initial Device Identifier):
   - Factory-provisioned X.509 certificate
   - Never changes during device lifetime
   - Burned into hardware (ROM, OTP, secure element)

2. **LDevID** (Locally Significant Device Identifier):
   - Provisioned after deployment
   - Can be updated/renewed
   - Used for operational authentication

#### Open-Source Implementations

**strongSwan DevID Plugin** (Linux/OpenWrt):
```bash
# Install strongSwan with 802.1AR support
opkg install strongswan-mod-eap-tls strongswan-mod-x509

# Configure IDevID certificate
cat > /etc/swanctl/x509/idevid.pem <<EOF
-----BEGIN CERTIFICATE-----
[Device certificate from manufacturer]
-----END CERTIFICATE-----
EOF

# Use for IPsec authentication
conn device-to-cloud
    remote_addrs=cloud.example.com
    local {
        auth = pubkey
        certs = idevid.pem
    }
```

**TCG Reference Implementation**:
- [TCG DevID Resources](https://trustedcomputinggroup.org/resource/device-identifier-composition-engine/)
- Reference code for certificate provisioning

#### Integration Example

```c
// Retrieve IDevID certificate from secure storage
int get_device_identity(X509 **cert) {
    BIO *bio = BIO_new_file("/sys/firmware/devid/certificate.pem", "r");
    if (!bio) {
        return -1;
    }

    *cert = PEM_read_bio_X509(bio, NULL, NULL, NULL);
    BIO_free(bio);

    if (*cert) {
        // Verify certificate chain
        verify_certificate_chain(*cert, ca_cert);
        return 0;
    }

    return -1;
}

// Authenticate device to cloud service
int authenticate_device_to_cloud(void) {
    X509 *idevid_cert;
    EVP_PKEY *private_key;

    // Get factory-provisioned IDevID
    get_device_identity(&idevid_cert);

    // Private key stored in secure element/TPM
    private_key = load_secure_key(KEY_SLOT_IDEVID);

    // Use for TLS client authentication
    SSL_CTX_use_certificate(ssl_ctx, idevid_cert);
    SSL_CTX_use_PrivateKey(ssl_ctx, private_key);

    // Connect to cloud with mutual TLS
    return ssl_connect(cloud_server, ssl_ctx);
}
```

**Best For**: Enterprise network equipment, industrial IoT gateways, devices requiring network authentication

---

### ARM Platform Security Architecture (PSA) Certified Device Identity

**Standard**: [PSA Certified](https://www.psacertified.org/)
**Security Levels**: PSA Level 1 (software), Level 2 (hardware isolation), Level 3 (anti-tamper)
**Hardware**: ARM TrustZone-M (Cortex-M23/M33/M55/M85)

#### Key Components

1. **Trusted Firmware-M (TF-M)**: Secure firmware providing PSA services
2. **PSA Crypto API**: Standardized cryptographic operations
3. **PSA Initial Attestation**: Device identity and firmware measurement

#### Open-Source Stack

**Mbed TLS PSA Crypto** (https://github.com/Mbed-TLS/mbedtls):
```c
#include "psa/crypto.h"

// Initialize PSA Crypto subsystem
psa_status_t init_device_identity(void) {
    psa_crypto_init();

    // Generate or retrieve device attestation key
    psa_key_id_t key_id;
    psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;

    psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_SIGN_HASH);
    psa_set_key_algorithm(&attributes, PSA_ALG_ECDSA(PSA_ALG_SHA_256));
    psa_set_key_type(&attributes, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
    psa_set_key_bits(&attributes, 256);
    psa_set_key_lifetime(&attributes,
                          PSA_KEY_LIFETIME_FROM_PERSISTENCE_AND_LOCATION(
                              PSA_KEY_PERSISTENCE_DEFAULT,
                              PSA_KEY_LOCATION_LOCAL_STORAGE));

    // Key stored in TrustZone secure storage
    return psa_generate_key(&attributes, &key_id);
}

// PSA Initial Attestation: Prove device identity and firmware state
psa_status_t attest_device(uint8_t *challenge, size_t challenge_len,
                            uint8_t *token, size_t *token_len) {
    // Generate attestation token (JWT with firmware measurements)
    return psa_initial_attest_get_token(challenge, challenge_len,
                                         token, *token_len, token_len);
}
```

**Trusted Firmware-M (TF-M)** (https://github.com/TrustedFirmwareM/trusted-firmware-m):
```c
// TF-M provides secure services to non-secure world
// Device identity service example

#include "tfm_platform_api.h"
#include "tfm_attest_api.h"

// Get unique device ID (Implementation Defined)
enum tfm_plat_err_t get_device_id(uint8_t *id, size_t *id_len) {
    return tfm_plat_get_implementation_id(id, id_len);
}

// Get attestation token signed by device key
enum tfm_status_e create_attestation(const uint8_t *challenge,
                                      size_t challenge_size,
                                      uint8_t *token,
                                      size_t *token_size) {
    return tfm_attest_get_token(challenge, challenge_size,
                                 token, *token_size, token_size);
}
```

#### Supported Hardware

| Chip Family | Vendor | Open-Source Support | Cost Range |
|-------------|--------|---------------------|------------|
| STM32L5/U5 | STMicroelectronics | ✅ TF-M + Mbed TLS | $2-8 |
| nRF5340/nRF9160 | Nordic Semi | ✅ Zephyr RTOS PSA | $3-10 |
| LPC55S6x | NXP | ✅ TF-M reference | $2-6 |
| SAMA5D27 | Microchip | ✅ TrustZone-A | $8-15 |
| i.MX 8M | NXP | ✅ OP-TEE | $15-30 |

**Integration with Yocto/Linux**:
```bitbake
# meta-arm layer for TF-M integration
MACHINE = "stm32mp15"
DISTRO_FEATURES:append = " optee"

# In local.conf
IMAGE_INSTALL:append = " optee-client optee-os"
IMAGE_INSTALL:append = " trusted-firmware-m"

# Device tree configuration for TrustZone
KERNEL_DEVICETREE = "stm32mp157c-dk2-scmi.dtb"
```

**Best For**: ARM Cortex-M microcontrollers, cost-sensitive IoT devices, battery-powered sensors

---

### TPM 2.0 Device Attestation (TCG Standard)

**Standard**: [TCG TPM 2.0](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
**Purpose**: Hardware root of trust for cryptographic operations and attestation
**Open-Source**: Complete software stack available

#### TPM 2.0 Architecture

```
┌─────────────────────────────────────────┐
│         Applications                     │
│  (Device Auth, Disk Encryption, etc.)   │
└─────────────────────────────────────────┘
              ▲
              │ TPM 2.0 API
              ▼
┌─────────────────────────────────────────┐
│      tpm2-tools / tpm2-tss              │
│   (Open-source TPM Software Stack)      │
└─────────────────────────────────────────┘
              ▲
              │ /dev/tpm0
              ▼
┌─────────────────────────────────────────┐
│       Linux Kernel TPM Driver           │
└─────────────────────────────────────────┘
              ▲
              │ SPI/I2C/LPC
              ▼
┌─────────────────────────────────────────┐
│         TPM 2.0 Hardware Chip           │
│  (Infineon SLB9670, Nuvoton NPCT75x)   │
└─────────────────────────────────────────┘
```

#### Open-Source TPM Stack

**tpm2-software** (https://github.com/tpm2-software):
```bash
# Install on embedded Linux
apt-get install tpm2-tools libtpm2-tss-dev

# Read TPM device identity (Endorsement Key certificate)
tpm2_nvread 0x01C00002 -o ek_cert.der

# Convert to PEM
openssl x509 -inform der -in ek_cert.der -out ek_cert.pem

# Create Attestation Identity Key (AIK) for device authentication
tpm2_createak -C ek.ctx -G rsa -g sha256 -s rsassa \
              -c ak.ctx -u ak.pub -n ak.name

# Generate attestation quote (prove firmware state)
tpm2_quote -c ak.ctx -l sha256:0,1,2,3 -q challenge.bin -m quote.bin -s signature.bin
```

**Using TPM for Device Authentication** (C):
```c
#include <tss2/tss2_esys.h>
#include <tss2/tss2_mu.h>

// Initialize TPM context
ESYS_CONTEXT* init_tpm(void) {
    ESYS_CONTEXT *esys_ctx;
    TSS2_RC rc = Esys_Initialize(&esys_ctx, NULL, NULL);
    if (rc != TSS2_RC_SUCCESS) {
        return NULL;
    }
    return esys_ctx;
}

// Read device Endorsement Key (EK) certificate
int read_device_identity(ESYS_CONTEXT *ctx, TPM2B_PUBLIC *ek_pub) {
    ESYS_TR ek_handle;

    // Read EK from TPM NVRAM (NV Index 0x01C00002)
    TSS2_RC rc = Esys_NV_Read(ctx,
                              ESYS_TR_RH_OWNER,
                              0x01C00002,  // EK certificate NV index
                              ESYS_TR_PASSWORD, ESYS_TR_NONE, ESYS_TR_NONE,
                              sizeof(*ek_pub), 0,
                              (TPM2B_MAX_NV_BUFFER **)&ek_pub, NULL);

    return (rc == TSS2_RC_SUCCESS) ? 0 : -1;
}

// Sign data with Attestation Key for device authentication
int sign_with_device_key(ESYS_CONTEXT *ctx,
                          ESYS_TR ak_handle,
                          uint8_t *data, size_t data_len,
                          TPM2B_SIGNATURE *signature) {
    TPM2B_DIGEST digest = {.size = 32};

    // Hash data (TPM can do this internally)
    TSS2_RC rc = Esys_Hash(ctx, ESYS_TR_NONE, ESYS_TR_NONE, ESYS_TR_NONE,
                           (TPM2B_MAX_BUFFER *)data, TPM2_ALG_SHA256,
                           ESYS_TR_RH_NULL, &digest, NULL);

    // Sign with AK (Attestation Key)
    rc = Esys_Sign(ctx, ak_handle,
                   ESYS_TR_PASSWORD, ESYS_TR_NONE, ESYS_TR_NONE,
                   &digest, NULL, NULL, signature);

    return (rc == TSS2_RC_SUCCESS) ? 0 : -1;
}

// Remote attestation: Prove device state to cloud
int remote_attestation(ESYS_CONTEXT *ctx,
                        uint8_t *challenge, size_t challenge_len) {
    TPM2B_ATTEST attest_data;
    TPM2B_SIGNATURE signature;

    // Generate quote (signed PCR values + nonce)
    TPML_PCR_SELECTION pcr_select = {
        .count = 1,
        .pcrSelections[0] = {
            .hash = TPM2_ALG_SHA256,
            .sizeofSelect = 3,
            .pcrSelect = {0xFF, 0xFF, 0xFF}  // PCRs 0-23
        }
    };

    TPM2B_DATA nonce = {.size = challenge_len};
    memcpy(nonce.buffer, challenge, challenge_len);

    TSS2_RC rc = Esys_Quote(ctx, attestation_key_handle,
                             ESYS_TR_PASSWORD, ESYS_TR_NONE, ESYS_TR_NONE,
                             &nonce, NULL, &pcr_select,
                             &attest_data, &signature);

    // Send attestation to verifier
    send_to_verifier(&attest_data, &signature);

    return (rc == TSS2_RC_SUCCESS) ? 0 : -1;
}
```

#### TPM Integration with OpenWrt/Yocto

**OpenWrt**:
```bash
# opkg package list
opkg install tpm2-tools tpm2-tss kmod-tpm-tis-spi

# Enable TPM kernel module
modprobe tpm_tis_spi

# Verify TPM is detected
ls /dev/tpm*
```

**Yocto**:
```bitbake
# In local.conf or image recipe
IMAGE_INSTALL:append = " tpm2-tools tpm2-abrmd tpm2-tss"
KERNEL_FEATURES:append = " features/tpm/tpm.scc"

# Device tree overlay for SPI-connected TPM
KERNEL_DEVICETREE:append = " overlays/tpm-slb9670-spi0.dtbo"
```

**Best For**: Gateways, industrial PCs, automotive ECUs, devices requiring attestation

---

### DICE (Device Identifier Composition Engine) - TCG Standard

**Standard**: [TCG DICE](https://trustedcomputinggroup.org/work-groups/dice-architectures/)
**Purpose**: Layered device identity based on firmware measurements
**Advantages**: Hardware-agnostic, can be implemented in ROM or first-stage bootloader

#### DICE Architecture

```
Hardware UDS (Unique Device Secret) - Burned in OTP/eFuses
    │
    ├─> Layer 0 (ROM): Measure bootloader → CDI₀
    │
    ├─> Layer 1 (Bootloader): Measure firmware → CDI₁
    │
    ├─> Layer 2 (Firmware): Measure app → CDI₂
    │
    └─> Layer N: Device Certificate

CDI = Compound Device Identifier (derived from measurements)
```

#### Open-Source Implementations

**OpenDICE** (Google, https://github.com/google/open-dice):
```c
#include "dice/dice.h"

// UDS: Unique Device Secret (256-bit) burned in OTP
uint8_t uds[DICE_UDS_SIZE] = {/* factory programmed */};

// Derive CDI from firmware measurement
DiceResult derive_cdi(const uint8_t *firmware, size_t firmware_size,
                       uint8_t cdi[DICE_CDI_SIZE]) {
    uint8_t hash[DICE_HASH_SIZE];

    // Measure firmware
    DiceHash(NULL, firmware, firmware_size, hash);

    // Derive CDI: HMAC-SHA512(UDS, hash)
    return DiceDeriveCdiPrivateKeySeed(
        NULL,
        uds,           // Unique Device Secret
        hash,          // Firmware measurement
        cdi            // Output: Compound Device Identifier
    );
}

// Generate device certificate from CDI
DiceResult generate_device_cert(const uint8_t cdi[DICE_CDI_SIZE],
                                 uint8_t *cert_buf, size_t *cert_size) {
    uint8_t public_key[DICE_PUBLIC_KEY_SIZE];
    uint8_t private_key[DICE_PRIVATE_KEY_SIZE];

    // Derive keypair from CDI
    DiceKeypairFromSeed(NULL, cdi, public_key, private_key);

    // Generate X.509 certificate
    DiceGenerateCertificate(
        NULL,
        subject_name,      // "Device Serial 12345"
        issuer_name,       // "Manufacturer CA"
        public_key,
        private_key,
        cert_buf,
        cert_size
    );

    return kDiceResultOk;
}
```

**Project Mu DICE** (Microsoft, open-source):
```c
// Used in Azure Sphere and Surface devices
#include "DiceCore.h"

// Layered DICE implementation
DICE_RESULT DiceLayeredFlow(
    const uint8_t *uds,
    uint8_t *current_cdi,
    const MeasurementData *measurements,
    size_t num_layers
) {
    for (size_t i = 0; i < num_layers; i++) {
        uint8_t next_cdi[DICE_CDI_SIZE];

        // Measure next layer
        DICE_RESULT result = DiceMainFlow(
            current_cdi,
            measurements[i].code_descriptor,
            measurements[i].code_hash,
            measurements[i].config_descriptor,
            measurements[i].config_hash,
            measurements[i].authority_descriptor,
            measurements[i].authority_hash,
            next_cdi,
            &layer_cert[i]
        );

        if (result != kDiceResultOk) {
            return result;
        }

        memcpy(current_cdi, next_cdi, DICE_CDI_SIZE);
    }

    return kDiceResultOk;
}
```

#### Integration with U-Boot

```c
// U-Boot first-stage DICE implementation
// File: board/vendor/board/dice.c

#include <common.h>
#include <dice.h>

// Read UDS from eFuses
static int read_uds(uint8_t uds[32]) {
    // Platform-specific: read from OTP/eFuses
    return fuse_read(FUSE_BANK_UDS, 0, (u32 *)uds, 8);
}

// Measure next boot stage (U-Boot SPL → U-Boot proper)
int dice_measure_next_stage(void *load_addr, size_t load_size) {
    uint8_t uds[DICE_UDS_SIZE];
    uint8_t cdi[DICE_CDI_SIZE];

    // Get device secret
    read_uds(uds);

    // Measure loaded firmware
    uint8_t hash[32];
    sha256_csum((unsigned char *)load_addr, load_size, hash);

    // Derive CDI
    DiceDeriveCdiPrivateKeySeed(NULL, uds, hash, cdi);

    // Store CDI in secure SRAM for next stage
    memcpy((void *)CONFIG_DICE_CDI_ADDR, cdi, sizeof(cdi));

    return 0;
}
```

**Best For**: Supply chain security, firmware integrity validation, boot attestation

---

### Microchip Secure Elements (ATECC608A/B)

**Hardware**: Low-cost I2C cryptographic coprocessor (~$0.50-1 in volume)
**Open-Source**: Complete driver stack available
**Pre-provisioned**: Factory-burned unique serial number + ECC-P256 keypair

#### Features

- **Hardware Key Storage**: Private keys never exposed to MCU
- **ECDH/ECDSA**: ECC-P256 operations in hardware
- **Secure Boot**: SHA-256 HMAC for firmware verification
- **Pre-configured**: Can be ordered pre-provisioned for AWS IoT, Azure IoT Hub

#### Open-Source Driver: cryptoauthlib

**cryptoauthlib** (https://github.com/MicrochipTech/cryptoauthlib):
```c
#include "cryptoauthlib.h"

// Initialize ATECC608
ATCAIfaceCfg cfg = {
    .iface_type = ATCA_I2C_IFACE,
    .devtype = ATECC608,
    .atcai2c.slave_address = 0xC0,
    .atcai2c.bus = 1,
    .atcai2c.baud = 400000,
};

ATCA_STATUS init_secure_element(void) {
    return atcab_init(&cfg);
}

// Read factory-provisioned device serial number
ATCA_STATUS get_device_serial(uint8_t serial[9]) {
    return atcab_read_serial_number(serial);
}

// Sign challenge with device private key (slot 0)
ATCA_STATUS sign_challenge(const uint8_t *challenge, size_t len,
                            uint8_t signature[64]) {
    uint8_t digest[32];

    // Hash challenge
    atcab_hw_sha2_256(challenge, len, digest);

    // Sign with private key in slot 0 (key never leaves chip)
    return atcab_sign(0, digest, signature);
}

// Authenticate device to AWS IoT Core
int authenticate_to_aws_iot(void) {
    uint8_t serial[9];
    uint8_t signature[64];
    uint8_t device_cert[512];
    size_t cert_size;

    // Get device serial
    atcab_read_serial_number(serial);

    // Read device certificate (pre-provisioned by Microchip)
    atcab_read_zone(ATCA_ZONE_DATA, 10, 0, 0, device_cert, 72);

    // TLS client authentication using ATECC608 for signing
    // (MbedTLS integration via cryptoauthlib PKCS#11 interface)

    return mqtt_connect_aws("a3xxxxxx-ats.iot.us-east-1.amazonaws.com",
                             device_cert, serial);
}

// Secure firmware verification with ATECC608 HMAC
ATCA_STATUS verify_firmware_signature(const uint8_t *firmware, size_t size,
                                       const uint8_t *expected_hmac) {
    uint8_t calculated_hmac[32];

    // Calculate HMAC-SHA256 using key in slot 4
    atcab_sha_hmac(firmware, size, 4, calculated_hmac, SHA_MODE_TARGET_OUT_ONLY);

    // Constant-time comparison
    if (memcmp(calculated_hmac, expected_hmac, 32) == 0) {
        return ATCA_SUCCESS;
    }

    return ATCA_CHECKMAC_VERIFY_FAILED;
}
```

#### Integration with Embedded Linux

**Device Tree** (for Linux kernel I2C driver):
```dts
&i2c1 {
    status = "okay";

    atecc608a: crypto@60 {
        compatible = "atmel,atecc608a";
        reg = <0x60>;
        status = "okay";
    };
};
```

**OpenWrt Package**:
```bash
opkg install libcryptoauth libcryptoauth-utils

# List available slots
cryptoauth-util -b 1 -a 0x60 info

# Read serial number
cryptoauth-util -b 1 -a 0x60 serial
```

**Yocto Integration**:
```bitbake
# meta-cryptoauth layer
BBLAYERS += "/path/to/meta-cryptoauth"

IMAGE_INSTALL:append = " cryptoauthlib cryptoauthlib-dev"

# Application links against libcryptoauth
DEPENDS:append = " cryptoauthlib"
```

#### Cloud Platform Integration

| Cloud Provider | Support | Configuration |
|----------------|---------|---------------|
| **AWS IoT Core** | ✅ Native | Pre-provisioned ATECC608A-MAHAW-T |
| **Azure IoT Hub** | ✅ Native | Pre-provisioned ATECC608A-MAHDA-T |
| **Google Cloud IoT** | ✅ Manual | Use ECDSA P-256 cert from slot 0 |

**Best For**: Cost-sensitive consumer IoT, battery-powered devices, cloud-connected sensors

---

### NXP EdgeLock SE050 Secure Element

**Hardware**: High-security element with GlobalPlatform API
**Certifications**: Common Criteria EAL 6+ (augmented with AVA_VAN.5), FIPS 140-2 Level 3
**Open-Source**: SE05X middleware fully open-source

#### Features

- **Multiple Crypto Algorithms**: RSA 4096, ECC (NIST, Brainpool, Ed25519), AES-256
- **Secure Storage**: 50KB for keys, certificates, and data
- **Applet Support**: JavaCard 3.0.4 for custom security applications
- **Post-Quantum Ready**: Firmware updates for PQC algorithms

#### Open-Source Middleware

**SE05X Plug & Trust Middleware** (https://github.com/NXPPlugNTrust):
```c
#include "se05x_APDU.h"
#include "sm_types.h"

// Initialize SE050 secure element
sss_status_t init_se050(sss_session_t *session) {
    sss_status_t status;

    // Open session with SE050 over I2C
    status = sss_session_open(session, kType_SSS_SE_SE05x,
                               0, kSSS_ConnectionType_Plain, NULL);

    return status;
}

// Generate device key pair in SE050
sss_status_t generate_device_keypair(sss_session_t *session,
                                      uint32_t key_id) {
    sss_object_t key_obj;
    sss_key_store_t *keystore = &session->ks;

    // Allocate key object
    sss_key_object_init(&key_obj, keystore);
    sss_key_object_allocate_handle(&key_obj, key_id,
                                     kSSS_KeyPart_Pair,
                                     kSSS_CipherType_EC_NIST_P,
                                     256, kKeyObject_Mode_Persistent);

    // Generate ECC P-256 key pair (private key never leaves SE050)
    sss_asymmetric_context_t asym_ctx;
    sss_asymmetric_context_init(&asym_ctx, session,
                                 &key_obj, kAlgorithm_SSS_ECDSA_SHA256,
                                 kMode_SSS_Sign);

    return sss_key_store_generate_key(keystore, &key_obj, 256, NULL);
}

// Sign data with device key in SE050
sss_status_t sign_with_device_key(sss_session_t *session,
                                    uint32_t key_id,
                                    const uint8_t *data, size_t data_len,
                                    uint8_t *signature, size_t *sig_len) {
    sss_object_t key_obj;
    sss_asymmetric_t asym_ctx;
    uint8_t digest[32];

    // Hash data (can be done in SE050 or externally)
    mbedtls_sha256(data, data_len, digest, 0);

    // Get key reference
    sss_key_object_init(&key_obj, &session->ks);
    sss_key_object_get_handle(&key_obj, key_id);

    // Initialize signing context
    sss_asymmetric_context_init(&asym_ctx, session, &key_obj,
                                 kAlgorithm_SSS_ECDSA_SHA256,
                                 kMode_SSS_Sign);

    // Sign digest with private key in SE050
    return sss_asymmetric_sign_digest(&asym_ctx, digest, sizeof(digest),
                                       signature, sig_len);
}

// Store device certificate in SE050
sss_status_t store_device_certificate(sss_session_t *session,
                                       uint32_t cert_id,
                                       const uint8_t *cert, size_t cert_len) {
    sss_object_t cert_obj;

    sss_key_object_init(&cert_obj, &session->ks);
    sss_key_object_allocate_handle(&cert_obj, cert_id,
                                     kSSS_KeyPart_Default,
                                     kSSS_CipherType_Binary,
                                     cert_len, kKeyObject_Mode_Persistent);

    return sss_key_store_set_key(&session->ks, &cert_obj,
                                  cert, cert_len, cert_len * 8, NULL, 0);
}
```

#### Integration with Matter (Smart Home Standard)

**Matter Device Attestation** using SE050:
```c
// Matter uses Device Attestation Certificate (DAC) stored in SE050
#include "credentials/DeviceAttestationCredsProvider.h"

class SE050AttestationProvider : public chip::Credentials::DeviceAttestationCredentialsProvider {
public:
    CHIP_ERROR GetCertificationDeclaration(MutableByteSpan &out_span) override {
        // Read CD from SE050 secure storage
        return read_from_se050(CD_OBJECT_ID, out_span);
    }

    CHIP_ERROR GetFirmwareInformation(MutableByteSpan &out_span) override {
        // Return firmware version and hash
        return get_firmware_info(out_span);
    }

    CHIP_ERROR GetDeviceAttestationCert(MutableByteSpan &out_span) override {
        // Read DAC from SE050
        return read_from_se050(DAC_OBJECT_ID, out_span);
    }

    CHIP_ERROR SignWithDeviceAttestationKey(const ByteSpan &message,
                                              MutableByteSpan &out_span) override {
        // Sign with private key in SE050 (never exposed)
        size_t sig_len = out_span.size();
        sss_status_t status = sign_with_device_key(&se050_session,
                                                     DAC_KEY_ID,
                                                     message.data(), message.size(),
                                                     out_span.data(), &sig_len);
        out_span.reduce_size(sig_len);
        return (status == kStatus_SSS_Success) ? CHIP_NO_ERROR : CHIP_ERROR_INTERNAL;
    }
};
```

**Yocto Integration**:
```bitbake
# meta-nxp-se05x layer
BBLAYERS += "/path/to/meta-nxp-se05x"

IMAGE_INSTALL:append = " se05x-middleware imx-se050-plugin"

# Enable I2C interface for SE050
MACHINE_FEATURES:append = " se05x"
```

**Best For**: Smart home devices (Matter/Thread), industrial IoT, payment terminals, high-security applications

---

### Comparison Table: Hardware Device Identity Solutions

| Solution | Standard | Hardware Cost | Crypto | Open-Source | Best Use Case | Cloud Integration |
|----------|----------|---------------|--------|-------------|---------------|-------------------|
| **TPM 2.0** | TCG | $1-5 | RSA, ECC | ✅ tpm2-tools | Enterprise, gateways, attestation | AWS Nitro, Azure DCsv3 |
| **ATECC608** | - | $0.50-1 | ECC-P256 | ✅ cryptoauthlib | Cost-sensitive IoT | AWS IoT, Azure IoT Hub |
| **EdgeLock SE050** | GlobalPlatform | $2-4 | RSA, ECC, Ed25519 | ✅ SE05X MW | High-security IoT, Matter | Matter-certified clouds |
| **ARM PSA (TF-M)** | PSA Certified | SoC-integrated | ECC, Chacha20 | ✅ TrustedFirmware-M | ARM Cortex-M devices | Azure IoT, AWS FreeRTOS |
| **DICE** | TCG | Software/ROM | SHA-256, HMAC | ✅ OpenDICE | Boot attestation | Google Cloud, Azure Sphere |
| **IEEE 802.1AR** | IEEE | Varies | RSA, ECC | ✅ strongSwan | Network equipment | Cisco ISE, Aruba ClearPass |

---

### Cisco Secure Unique Device Identifier (SUDI) - Enterprise Example

While not open-source hardware, Cisco SUDI demonstrates **production deployment of IEEE 802.1AR** in enterprise networking equipment and serves as a reference implementation.

**What is SUDI?**

- **IEEE 802.1AR compliant**: Factory-installed X.509 certificate
- **Manufacturer-signed**: Chain of trust to Cisco root CA
- **Hardware-protected**: Private key stored in Cisco Trust Anchor module
- **Use Cases**:
  - Zero-touch provisioning (Cisco DNA Center)
  - Network admission control (Cisco ISE)
  - Secure device onboarding

**SUDI Certificate Structure**:
```
Subject: serialNumber=PID:C9300-48P SN:FCW1234A567
Issuer: CN=Cisco Manufacturing CA, O=Cisco Systems
X509v3 Extended Key Usage: TLS Web Client Authentication
```

**Integration Example** (for understanding 802.1AR in practice):
```c
// Reading SUDI certificate from Cisco device (for reference)
// This demonstrates what an 802.1AR implementation provides

// On Cisco IOS-XE:
// Router# show crypto pki certificates CISCO_IDEVID_SUDI
//
// Certificate:
//   Subject: serialNumber=PID:ISR4451/K9 SN:FDO1234B5CD
//   Issuer: CN=Cisco Manufacturing CA SHA2
//   Validity: Not Before: Jan 1 00:00:00 2020 GMT
//             Not After : Dec 31 23:59:59 2029 GMT

// Equivalent open-source approach with strongSwan + 802.1AR:
X509 *load_idevid_certificate(const char *cert_path) {
    BIO *bio = BIO_new_file(cert_path, "r");
    X509 *cert = PEM_read_bio_X509(bio, NULL, NULL, NULL);
    BIO_free(bio);

    // Verify manufacturer signature
    X509_STORE *store = X509_STORE_new();
    X509_STORE_add_cert(store, manufacturer_ca_cert);

    X509_STORE_CTX *ctx = X509_STORE_CTX_new();
    X509_STORE_CTX_init(ctx, store, cert, NULL);

    int verify_result = X509_verify_cert(ctx);
    // verify_result == 1: Valid certificate chain

    X509_STORE_CTX_free(ctx);
    X509_STORE_free(store);

    return cert;
}
```

**Lessons for Open-Source Implementations**:
1. Factory-provision certificates during manufacturing
2. Chain trust to manufacturer CA
3. Include serial number and product ID in certificate subject
4. Use hardware protection for private keys
5. Support standard protocols (802.1X, TLS client auth)

**Open-Source Alternative**: strongSwan + ATECC608/SE050 + 802.1AR provides similar functionality for custom hardware.

---

## Practical Implementation Guide

### Choosing the Right Device Identity Solution

**Decision Flowchart**:
```
Do you need attestation (prove firmware state)?
├─ YES → TPM 2.0 or DICE
└─ NO ↓

What is your hardware platform?
├─ ARM Cortex-M → ARM PSA (TF-M)
├─ ARM Cortex-A / x86 → TPM 2.0
├─ Generic MCU → ATECC608 or SE050
└─ Network Equipment → 802.1AR (SUDI-style)

What is your security level requirement?
├─ EAL 6+, FIPS 140-2 → EdgeLock SE050
├─ PSA Level 2+ → TrustZone + TF-M
├─ Basic hardware protection → ATECC608
└─ Software-only → DICE (boot ROM implementation)

What is your budget per unit?
├─ < $1 → ATECC608
├─ $1-3 → ARM PSA (SoC-integrated)
├─ $2-5 → EdgeLock SE050 or TPM 2.0
└─ > $5 → Multiple elements (TPM + SE)
```

### Sample Integration: ATECC608 + Yocto + AWS IoT

**Step 1: Hardware Connection** (I2C)
```
Microprocessor (I2C Master)
    │
    ├─ SCL ──────────────── ATECC608 SCL
    ├─ SDA ──────────────── ATECC608 SDA
    ├─ 3.3V ─────────────── ATECC608 VCC
    └─ GND ──────────────── ATECC608 GND
```

**Step 2: Device Tree Configuration**
```dts
// arch/arm/boot/dts/myboard.dts
&i2c1 {
    atecc608a: crypto@60 {
        compatible = "atmel,atecc608a";
        reg = <0x60>;
    };
};
```

**Step 3: Yocto Configuration**
```bitbake
# conf/local.conf
IMAGE_INSTALL:append = " cryptoauthlib aws-iot-device-sdk-embedded-c"

# Kernel configuration
KERNEL_FEATURES:append = " features/i2c/i2c.scc"
```

**Step 4: Device Provisioning** (one-time, during manufacturing)
```bash
#!/bin/bash
# provision_device.sh - Run at factory

# Lock configuration zone (one-time operation, irreversible)
cryptoauth-util -b 1 -a 0x60 lock-config

# Write device certificate to data slots
cryptoauth-util -b 1 -a 0x60 write-cert --slot 10 device_cert.pem

# Lock data zone
cryptoauth-util -b 1 -a 0x60 lock-data
```

**Step 5: Application Code**
```c
// main.c - Device authentication to AWS IoT
#include "cryptoauthlib.h"
#include "aws_iot_mqtt_client.h"

int main(void) {
    // Initialize ATECC608
    atcab_init(&cfg_atecc608a_i2c_default);

    // Read device serial
    uint8_t serial[9];
    atcab_read_serial_number(serial);

    // Configure AWS IoT connection
    AWS_IoT_Client client;
    IoT_Client_Init_Params params = iotClientInitParamsDefault;
    params.pHostURL = "a3xxxxx-ats.iot.us-east-1.amazonaws.com";
    params.port = 8883;

    // Use ATECC608 for TLS (via PKCS#11 interface)
    params.pDeviceCertLocation = "pkcs11:object=device-cert";
    params.pDevicePrivateKeyLocation = "pkcs11:object=device-key";

    // Connect to AWS IoT Core
    aws_iot_mqtt_init(&client, &params);
    aws_iot_mqtt_connect(&client, &connectParams);

    // Publish telemetry with device identity
    char topic[128];
    snprintf(topic, sizeof(topic), "device/%s/telemetry", serial);
    aws_iot_mqtt_publish(&client, topic, strlen(topic), &paramsQOS1);

    return 0;
}
```

---

## Security Best Practices

### Device Identity Lifecycle

1. **Manufacturing**:
   - Provision unique credentials in secure facility
   - Burn keys into OTP/eFuses or secure elements
   - Generate and store device certificates
   - Document chain of custody

2. **Deployment**:
   - Verify device identity during onboarding
   - Establish trust with backend services
   - Rotate LDevID certificates (802.1AR)
   - Implement attestation verification

3. **Operations**:
   - Monitor for cloned devices (duplicate identities)
   - Implement certificate expiry management
   - Log authentication events for audit
   - Detect and respond to identity theft

4. **Decommissioning**:
   - Revoke device certificates
   - Update CRLs (Certificate Revocation Lists)
   - Securely wipe device identity data
   - Remove from device registry

### Zero Trust Integration

Device identity enables Zero Trust architectures:

```c
// Zero Trust authentication flow
int zero_trust_device_authentication(void) {
    // 1. Device proves identity with hardware credential
    X509 *device_cert = load_device_certificate();

    // 2. Backend verifies certificate chain
    if (!verify_certificate(device_cert, trusted_ca)) {
        return -1;  // Reject untrusted device
    }

    // 3. Device provides attestation (prove firmware state)
    uint8_t attestation_token[512];
    create_attestation_token(attestation_token);

    // 4. Backend verifies attestation against known-good measurements
    if (!verify_attestation(attestation_token, expected_pcr_values)) {
        return -1;  // Reject compromised device
    }

    // 5. Grant conditional access (time-limited, scoped)
    access_token = issue_access_token(device_cert, ACCESS_SCOPE_TELEMETRY,
                                       VALIDITY_24H);

    // 6. Continuous verification (re-attest periodically)
    schedule_reattestation(EVERY_6_HOURS);

    return 0;  // Authenticated and authorized
}
```

---

## Additional References <a id="additional-references"></a>

### Device Identity Standards
* [IEEE 802.1AR-2018: Secure Device Identity](https://standards.ieee.org/standard/802_1AR-2018.html)
* [TCG DICE: Device Identifier Composition Engine](https://trustedcomputinggroup.org/work-groups/dice-architectures/)
* [ARM Platform Security Architecture (PSA)](https://www.psacertified.org/)
* [TCG TPM 2.0 Library Specification](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
* [GlobalPlatform Secure Element Specifications](https://globalplatform.org/specs-library/)

### Open-Source Projects
* [Trusted Firmware-M (TF-M)](https://www.trustedfirmware.org/projects/tf-m/)
* [Mbed TLS PSA Crypto API](https://github.com/Mbed-TLS/mbedtls)
* [tpm2-software (TPM 2.0 Stack)](https://github.com/tpm2-software)
* [cryptoauthlib (Microchip)](https://github.com/MicrochipTech/cryptoauthlib)
* [SE05X Plug & Trust (NXP)](https://github.com/NXPPlugNTrust/nano-package)
* [OpenDICE (Google)](https://github.com/google/open-dice)
* [strongSwan VPN](https://www.strongswan.org/)

### Cloud Platform Guides
* [AWS IoT Device Provisioning](https://docs.aws.amazon.com/iot/latest/developerguide/device-certs-your-own.html)
* [Azure IoT Device Provisioning Service](https://docs.microsoft.com/en-us/azure/iot-dps/)
* [Google Cloud IoT Core Authentication](https://cloud.google.com/iot/docs/how-tos/credentials/keys)

### Compliance and Certifications
* [Common Criteria Portal](https://www.commoncriteriaportal.org/)
* [FIPS 140-2/140-3 Standards](https://csrc.nist.gov/projects/cryptographic-module-validation-program)
* [PSA Certified](https://www.psacertified.org/getting-certified/)

---

## User Identity Management (Original Content)

In addition to device identity, proper **user** identity management remains critical for embedded systems with human operators.

### User Account Security (Original Requirements)

The user identity requirements documented at the beginning of this chapter remain essential:

- No static default passwords across product lines
- Forced password changes upon device setup
- Separation of remote and local login accounts
- Account lockout thresholds for brute-force protection
- Session management best practices (HttpOnly cookies, session invalidation)
- Password complexity enforcement (10+ chars, mixed case, numbers, special chars)
- Certificate and key rotation policies

### References

**User Identity**:
* [NIST Special Publication 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)
* [FTC Charges D-Link Put Consumers' Privacy at Risk Due to the Inadequate Security of Its Computer Routers and Cameras](https://www.ftc.gov/news-events/press-releases/2017/01/ftc-charges-d-link-put-consumers-privacy-risk-due-inadequate)
* [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
* [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/) (Page 26-31)
* [SB-327 Information privacy: connected devices](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=201720180SB327)

