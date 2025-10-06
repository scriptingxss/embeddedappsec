# Firmware Updates and Cryptographic Signatures

Ensure robust update mechanisms utilize cryptographically signed firmware images upon download and when applicable, for updating functions pertaining to third party software. Cryptographic signature allows for verification that files have not been modified or otherwise tampered with since the developer created and signed them. The signing and verification process uses public-key cryptography and it is difficult to forge a digital signature \(e.g. PGP signature\) without first gaining access to the private key. In the event a private key is compromised, developers of the software must revoke the compromised key and will need to re-sign all previous firmware releases with the new key.

**Verifying a kernel image signature Example:**

Downloading the kernel images

```bash
wget [https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.6.6.tar.xz](https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.6.6.tar.xz)

wget [https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.6.6.tar.sign](https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.6.6.tar.sign)
```

**Download the public key from a PGP keyserver in order to verify the signature.**

```bash
# gpg2 --keyserver hkp://keys.gnupg.net --recv-keys 38DBBDC86092693E
gpg: /root/.gnupg/trustdb.gpg: trustdb created
gpg: key 38DBBDC86092693E: public key "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```

**Uncompressing and verifying the .tar firmware image against the signature:**

```bash
# xz -cd linux-4.6.6.tar.xz | gpg2 --verify linux-4.6.6.tar.sign -
gpg: Signature made Wed 10 Aug 2016 06:55:15 AM EDT
gpg:                using RSA key 38DBBDC86092693E
gpg: Good signature from "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 647F 2865 4894 E3BD 4571  99BE 38DB BDC8 6092 693E
```

Notice the WARNING: This key is not certified with a trusted signature! You will now need to verify that the key used to sign the archive really does belong to the owner \(in our example, Greg Kroah-Hartman\). There are several ways you can do this:

1. Use the Kernel.org web of trust. This will require that you first locate the members of kernel.org in your area and sign their keys. Short of meeting the actual owner of the PGP key in real life, this is your best option to verify the validity of a PGP key signature.
2. Review the list of signatures on the developer's key by using `gpg --list-sigs`. Email as many people who have signed the key as possible, preferably at different organizations \(or at least different domains\). Ask them to confirm that they have signed the key in question. You should attach, at best, marginal trust to the responses you receive in this manner \(if you receive any\).
3. Use the following site to see trust paths from Linus Torvalds' key to the key used to sign the tarball: pgp.cs.uu.nl. Put Linus's key into the "from" field and the key you got in the output above into the "to" field. Normally, only Linus or people with Linus's direct signature will be in charge of releasing kernels. 

If you get "BAD signature"  
If at any time you see "BAD signature" output from `gpg --verify`, please check the following first:

1. Make sure that you are verifying the signature against the .tar version of the archive, not the compressed \(.tar.xz\) version.
2. Make sure the the downloaded file is correct and not truncated or otherwise corrupted.

**Demonstrating \#1 above, verifying a signature incorrectly Example**:

```bash
# gpg --verify linux-4.6.6.tar.sign linux-4.6.6.tar.xz 
gpg: Signature made Wed 10 Aug 2016 06:55:15 AM EDT
gpg:                using RSA key 38DBBDC86092693E
gpg: BAD signature from "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" [unknown]
```

**Verifying a signature correctly Example**:

```bash
# gpg --verify linux-4.6.6.tar.sign linux-4.6.6.tar
gpg: Signature made Wed 10 Aug 2016 06:55:15 AM EDT
gpg:                using RSA key 38DBBDC86092693E
gpg: Good signature from "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 647F 2865 4894 E3BD 4571  99BE 38DB BDC8 6092 693E
```

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

## Post-Quantum Cryptography Readiness for Secure Boot (2025+)

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
- **Kernel Hardening**: See [Chapter 6: Embedded Framework Hardening](6_embedded_framework_and_c-based_toolchain_hardeni.md) - Yocto kernel security
- **CVE Management**: See [Chapter 10: Third Party Components](10_third_party_code_and_components.md) - Yocto CVE checking

### Resources

- [Yocto Secure Boot Documentation](https://docs.yoctoproject.org/dev/dev-manual/securing-images.html#using-verified-boot)
- [U-Boot Verified Boot](https://source.denx.de/u-boot/u-boot/-/blob/master/doc/uImage.FIT/verified-boot.txt)
- [Automotive Grade Linux Secure Boot](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/07-system-hardening.html#hardened-boot)

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

