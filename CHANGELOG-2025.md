# OWASP Embedded Application Security - 2025 Changelog

**Version**: 2025.1

---

## Overview

This document tracks all updates made to the OWASP Embedded Application Security Best Practices guide during the 2025 modernization effort. The guide has been comprehensively updated to reflect current security standards, emerging threats (including quantum computing), and modern compliance requirements.

---

## Major Updates

### 1. OWASP IoT Ecosystem Integration

**New Document**: [Appendix A: OWASP IoT Ecosystem Alignment](owasp-iot-ecosystem-alignment.md)

All 13 chapters now include comprehensive mappings to the OWASP IoT Security ecosystem:

- **OWASP ISVS** (IoT Security Verification Standard): Testable security requirements
- **OWASP ISTG** (IoT Security Testing Guide): Testing procedures and methodologies
- **OWASP FSTM** (Firmware Security Testing Methodology): Firmware analysis stages
- **OWASP IoTGoat**: Hands-on vulnerable firmware for practice

**Impact**: Developers can now easily map embedded security best practices to testable requirements and training resources.

**Chapters Updated**: All 13 chapters

---

### 2. Device Compliance Frameworks

**New Document**: [Appendix B: Device Compliance Frameworks](device-compliance-frameworks.md)

Added comprehensive guidance for 9 major compliance frameworks:

| Framework | Market/Industry | Status |
|-----------|----------------|--------|
| EU Cyber Resilience Act | EU Consumer Products | Effective 2024 |
| ETSI EN 303 645 | EU IoT Devices | Active |
| IEC 62443 | Industrial/OT | Active |
| NIST IoT Cybersecurity | US Federal | Active |
| FDA Cybersecurity | Medical Devices | Active |
| ISO/SAE 21434 | Automotive | Active |
| UNECE WP.29 | Automotive (EU) | Active |
| Executive Order 14028 | US Federal | Active |
| CA SB-327 | California IoT | Active |

**Impact**: Embedded developers can identify compliance requirements by market and map them to implementation priorities.

---

### 3. Post-Quantum Cryptography (PQC) Readiness

Added comprehensive PQC guidance to protect against quantum computing threats:

#### Chapter 3: Firmware Updates and Cryptographic Signatures
**Section Added**: "Post-Quantum Cryptography Readiness for Secure Boot (2025+)"

- NIST PQC digital signature standards (ML-DSA, SLH-DSA, FN-DSA)
- Hybrid cryptography approach (Classical + PQC signatures)
- Bootloader implementation with hybrid signature verification
- Hardware acceleration requirements
- Storage considerations (13x signature size increase)
- Migration roadmap: 2025-2026 → 2026-2027 → 2028+

**Code Examples**: Hybrid ECDSA + ML-DSA signature verification in C

#### Chapter 4: Securing Sensitive Information
**Section Added**: "Post-Quantum Cryptography for Secure Storage (2025+)"

- NIST PQC key encapsulation mechanisms (ML-KEM/Kyber)
- Hybrid encryption (ECDH + ML-KEM)
- AES-256 upgrade guidance for quantum resistance
- TEE/HSM integration for PQC keys
- Storage overhead planning (~1-2KB per encrypted object)
- Progressive re-encryption migration strategy

**Code Examples**: Hybrid key encapsulation and AES-256-GCM encryption

#### Chapter 8: Transport Layer Security
**Section Added**: "Post-Quantum Cryptography for TLS (2025+)"

- Hybrid TLS 1.3 with PQC (ECDHE + ML-KEM)
- OpenSSL 3.x with liboqs integration
- wolfSSL PQC support
- Performance impact analysis (handshake: +2-4ms, bandwidth: +2.3KB)
- Certificate Authority PQC readiness timeline
- Testing tools and procedures

**Code Examples**: Embedded HTTPS server with hybrid PQC support

**Threat Addressed**: "Harvest Now, Decrypt Later" (HNDL) attacks - critical for devices with 10-20+ year lifecycles

---

### 4. Modern TLS & Cryptography Standards

**Chapter Updated**: [8_transport_layer_security.md](8_transport_layer_security.md)

- **TLS 1.3** (RFC 8446) now the recommended standard (TLS 1.2 minimum)
- Modern cipher suites: `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`
- Perfect Forward Secrecy (PFS) requirements
- Certificate lifecycle management and automation
- OCSP stapling and certificate transparency
- Compliance with PCI DSS 4.0, NIST SP 800-52 Rev. 2

---

### 5. Hardware Security Modules & Trusted Execution

**Chapter Updated**: [4_securing_sensitive_information.md](4_securing_sensitive_information.md)

Added comprehensive guidance on hardware-backed security:

**Trusted Execution Environments (TEE)**:
- ARM TrustZone implementation examples
- Intel SGX for x86 embedded systems
- TPM 2.0 integration patterns

**Secure Elements**:
- Microchip ATECC608 (I2C crypto chip)
- NXP EdgeLock SE050 (secure element)
- Infineon OPTIGA Trust M (automotive-grade)

**Code Examples**: C implementations for TEE key storage and secure crypto operations

---

### 6. Supply Chain Security & SBOM Modernization

**Chapter Updated**: [10_third_party_code_and_components.md](10_third_party_code_and_components.md)

**SBOM Standards**:
- **SPDX** (ISO/IEC 5962:2021) - standardized format
- **CycloneDX** - OWASP project for security-focused SBOMs

**Modern Tools**:
- **Syft**: Generate SBOMs from containers and filesystems
- **Trivy**: Comprehensive vulnerability scanner
- **Grype**: Vulnerability matching engine
- **OSV-Scanner**: Open Source Vulnerabilities scanner
- **Dependency-Track**: Continuous SBOM monitoring

**Supply Chain Security**:
- SLSA framework (Supply Chain Levels for Software Artifacts)
- Sigstore for artifact signing
- In-Toto for supply chain integrity

**Compliance**: Meets Executive Order 14028 and EU Cyber Resilience Act SBOM requirements

---

### 7. Command Injection Vulnerability Examples

**Chapter Updated**: [2_injection_prevention.md](2_injection_prevention.md)

Added 2 real-world embedded device injection vulnerability examples:

#### Example 1: OS Command Injection in Embedded Web Application
- **Vulnerable**: Network diagnostic feature using `system()` with unsanitized user input
- **Secure**: IP validation, allowlisting, `execve()` with argument array (no shell)
- **OWASP IoT Mapping**: ISVS V2.3.1, ISTG ISTG-UI-INFO-001, FSTM Stage 6

#### Example 2: Newline Command Injection in Configuration Files
- **Vulnerable**: Writing WiFi configuration without newline validation
- **Secure**: Comprehensive input validation with character allowlisting
- **OWASP IoT Mapping**: ISVS V2.3.2, ISTG ISTG-FW-CONF-001, FSTM Stage 4

Both examples include:
- Complete vulnerable code in C
- Complete secure implementation in C
- Mapping to OWASP IoT ecosystem (ISVS, ISTG, FSTM, IoTGoat)

---

### 8. Yocto Project Build System Security Integration

**Completed**: October 5, 2025

Added comprehensive Yocto Project security guidance across multiple chapters, making this the definitive resource for secure embedded Linux development using Yocto.

#### Chapter 6: Embedded Platform Security Hardening

**Section Added**: "Yocto Project Build System Security (2024-2025)" (~780 lines)

**Comprehensive Coverage**:
- **Yocto Scarthgap 5.0 LTS Security Foundation**: Core components, security enhancements
- **Compiler Security Flags**: Detailed explanation of security_flags.inc
  - Stack protection (`-fstack-protector-strong`)
  - FORTIFY_SOURCE (buffer overflow detection)
  - PIE/ASLR for address randomization
  - Format string protection
  - RELRO (read-only relocations)
  - GCC `-fhardened` flag (comprehensive protection)
- **Kernel Hardening in Yocto**:
  - Kernel configuration fragments (`.cfg` files)
  - KASLR (Kernel Address Space Layout Randomization)
  - KPTI (Kernel Page Table Isolation)
  - meta-security hardening configurations
  - Kernel hardening checker integration
  - Kernel module signing
- **Mandatory Access Control (MAC) Systems**:
  - SELinux, AppArmor, SMACK comparison table
  - Complete configuration examples for each MAC system
  - Policy creation and testing procedures
- **Integrity Measurement Architecture (IMA/EVM)**:
  - File integrity verification
  - Configuration and policy examples
- **Read-Only Root Filesystem with Overlays**:
  - OverlayFS configuration for selective writes
  - tmpfs for volatile data
- **Reproducible Builds**: Verification procedures
- **Making Images More Secure**: Official Yocto guide integration
- **Yocto Security Best Practices Checklist**: Complete 40-point checklist

**Code Examples**: 20+ BitBake configuration snippets, kernel config fragments, verification commands

#### Chapter 1: Buffer and Stack Overflow Protection

**Section Added**: "Yocto Project Compiler Hardening for Buffer Overflow Protection" (~190 lines)

**Coverage**:
- Stack protection configuration and flags
- FORTIFY_SOURCE for buffer overflow detection
- GCC `-fhardened` flag for maximum protection
- PIE/ASLR for exploit mitigation
- Verification tools (checksec, readelf)
- Buildroot vs. Yocto comparison table
- Per-recipe security flag customization
- Integration with Chapter 6 for comprehensive coverage

**Code Examples**: BitBake configurations, high-security recipe example, verification commands

#### Chapter 4: Securing Sensitive Information

**Section Added**: "TEE and Hardware Security Integration with Yocto Project" (~370 lines)

**Coverage**:
- **OP-TEE Integration** (ARM TrustZone):
  - Full configuration for meta-security layer
  - Machine-specific setup (Raspberry Pi 4, i.MX8)
  - Creating Trusted Application (TA) recipes
  - Complete TA code example
- **TPM 2.0 Support**:
  - Kernel configuration for TPM
  - TPM tools integration (tpm2-tss, tpm2-tools)
  - TPM application recipe examples
  - Key storage code example
- **Secure Element Integration** (I2C/SPI):
  - ATECC608, NXP EdgeLock SE050, OPTIGA Trust M
  - Device tree configuration
  - CryptoAuthLib recipe
  - Application integration examples
- **Hardware Security Best Practices**:
  - Secure boot chain with hardware root of trust
  - dm-crypt with TPM for encrypted storage
  - Development vs. production key provisioning
  - Attestation and remote provisioning
- **Verification and Testing**: Commands for OP-TEE, TPM, Secure Elements

**Code Examples**: 10+ BitBake recipes, C code for TEE/TPM/SE integration, device tree overlays

#### Chapter 3: Firmware Updates and Cryptographic Signatures

**Section Added**: "Yocto Project Secure Boot Implementation" (~180 lines)

**Coverage**:
- U-Boot Verified Boot configuration
- FIT image signing setup
- Key generation procedures
- Kernel FIT image signing
- Hardware root of trust integration (TPM, OP-TEE)
- Production vs. development key management
- Complete secure boot chain (ROM → SPL → U-Boot → Kernel → Rootfs → Runtime)
- Testing and verification procedures

**Code Examples**: BitBake secure boot configuration, key generation commands, verification tests

#### Chapter 10: Third Party Code and Components

**Already Complete**: Comprehensive Yocto CVE checking and SBOM generation content was already present in the guide. No updates needed.

**Existing Coverage**:
- CVE checking with `cve-check` class
- Advanced CVE filtering (kernel .config upload)
- Third-party vulnerability tools (meta-timesys, meta-cvescan)
- CI/CD CVE integration
- SPDX SBOM generation (2.2 and 3.0.1 formats)
- CycloneDX integration
- buildhistory package tracking
- SBOM validation and quality checks
- EU Cyber Resilience Act compliance
- Vulnerability scanning with SBOMs (Grype, OSV-Scanner, Trivy)

#### Cross-References

All Yocto sections include cross-references to related chapters:
- Chapter 1 → Chapter 6 (platform and toolchain hardening)
- Chapter 3 → Chapter 4 (hardware root of trust) and Chapter 6 (platform security)
- Chapter 4 → Chapter 3 (secure boot) and Chapter 6 (platform hardening)
- Chapter 6 → Chapter 10 (CVE checking and SBOM)

#### Resources Added

**Yocto Project Documentation**:
- Yocto Project Official Documentation - Making Images More Secure
- Yocto Project Security Wiki
- Yocto Security Hardening: Security Flags
- Yocto Kernel Development & Security Hardening
- meta-security Layer Index
- meta-selinux Git Repository
- Reproducible Builds in Yocto

**Hardware and Security Tools**:
- Kernel Hardening Checker (kernel config audit)
- OP-TEE in Yocto documentation
- TPM 2.0 Software Stack (TSS)
- Microchip ATECC608 Yocto Integration
- NXP i.MX Security (EdgeLock SE050)
- Infineon OPTIGA Trust M
- Timesys VigiShield (commercial CVE monitoring)

#### Impact

**For Developers**:
- Complete Yocto security configuration guidance in one place
- Specific BitBake configuration for all security features
- Production-ready code examples and recipes
- Clear comparison tables (Buildroot vs. Yocto, SELinux vs. AppArmor vs. SMACK)

**For Organizations**:
- Regulatory compliance guidance (FDA, automotive, industrial, EU CRA)
- Hardware security integration for high-security markets
- CVE management and SBOM generation for supply chain security
- Reproducible builds for supply chain verification

**Coverage**: The OWASP Embedded Application Security guide is now the most comprehensive open-source resource for secure Yocto-based embedded Linux development.

---

### 9. Chapter Content Expansion (Sessions 1-5)

**Completed**: October 2025

Comprehensive expansion of security guidance across all major chapters with modern best practices, detailed code examples, and regulatory compliance mapping.

#### Chapter 1: Buffer and Stack Overflow Protection
**Expansion**: +560 lines
- GCC security compilation flags (FORTIFY_SOURCE, stack protectors, PIE/ASLR)
- Kernel hardening parameters and memory protection
- Modern exploit mitigation techniques
- Yocto compiler hardening integration

#### Chapter 2: Injection Prevention
**Expansion**: +698 lines
- Real-world embedded device vulnerability examples
- OS command injection prevention patterns
- Input validation for resource-constrained environments
- Secure coding practices for embedded C/C++

#### Chapter 3: Firmware Updates and Cryptographic Signatures
**Expansion**: +992 lines
- Modern firmware update frameworks (SWUpdate, RAUC, Mender)
- U-Boot verified boot and FIT image signing
- Rollback protection and A/B update strategies
- Post-quantum cryptography readiness for secure boot

#### Chapter 4: Securing Sensitive Information
**Updates**: Hardware security and PQC enhancements
- Hardware-based security (TEE, secure elements, TPM 2.0)
- Post-quantum cryptography for secure storage
- Key management best practices

#### Chapter 6: Embedded Platform Security Hardening
**Restructuring**: Content deduplication and navigation improvements
- Removed duplicate U-Boot content (cross-referenced to Chapter 3)
- Removed outdated Buildroot screenshots
- Added Quick Navigation section with cross-chapter links
- Converted AGL-specific guidance to universal embedded practices
- Clarified platform-level focus vs compiler/bootloader details

#### Chapter 8: Transport Layer Security
**Expansion**: +865 lines
- Post-quantum cryptography for TLS (hybrid ECDHE + ML-KEM)
- Embedded TLS library comparisons (mbedTLS, wolfSSL, OpenSSL)
- Certificate management and mutual TLS (mTLS)
- TLS 1.3 configuration for resource-constrained devices
- Hardware crypto acceleration integration

#### Chapter 10: Third-Party Code and Components
**Expansion**: +797 lines
- Yocto CVE checking and vulnerability management
- Modern SBOM tools and generation (SPDX, CycloneDX)
- Compliance automation workflows
- CVE scanner integration (Grype, Trivy, Clair)
- SBOM differential analysis
- Dependency Track for continuous monitoring

**Total**: ~4,000 lines of new security guidance addressing modern threats, regulatory requirements (EU CRA, NTIA SBOM mandates), and quantum-resistant cryptography preparation.

**Impact**: Guide now provides production-ready code examples, automated workflows, and comprehensive coverage for devices with 10-20+ year lifecycles.

---

## Link Updates

### OWASP Wiki Migration (36 Links Updated)

All OWASP wiki links migrated from deprecated infrastructure to current URLs:

**Old Format**: `https://www.owasp.org/index.php/[Topic]`
**New Format**: `https://owasp.org/www-[project]/[topic]`

**Files Updated**:
1. 1_buffer_and_stack_overflow_protection.md (4 links)
2. 2_injection_prevention.md (3 links)
3. 3_firmware_updates_and_cryptographic_signatures.md (2 links)
4. 4_securing_sensitive_information.md (5 links)
5. 5identity_management.md (2 links)
6. 6_embedded_framework_and_c-based_toolchain_hardeni.md (6 links)
7. 7_usage_of_debugging_code_and_interfaces.md (1 link)
8. 8_transport_layer_security.md (4 links)
9. 9_usage_of_data_collection_and_storage_-_privacy.md (3 links)
10. 10_third_party_code_and_components.md (4 links)
11. threat-model.md (2 links)

### CERT Secure Coding Updates

Updated CERT Secure Coding references to new wiki domain:

**Old**: `https://www.securecoding.cert.org/*`
**New**: `https://wiki.sei.cmu.edu/confluence/display/seccode/*`

---

## Year Updates (2024 → 2025)

Updated 36 references across 8 files:

1. **README.md**: Project title and description
2. **SUMMARY.md**: Table of contents
3. **8_transport_layer_security.md**: TLS version recommendations, PCI DSS compliance dates
4. **10_third_party_code_and_components.md**: SBOM standards, tool versions
5. **project-roadmap.md**: Roadmap timeline updated to 2025-2026
6. **owasp-iot-ecosystem-alignment.md**: OWASP IoT project status
7. **device-compliance-frameworks.md**: Compliance framework effective dates
8. **threat-model.md**: Threat landscape updates

---

## File Corrections

### Empty Files Populated

**project-roadmap.md**:
- Added comprehensive 5-phase roadmap (2025-2026)
- Defined deliverables, timeline, and success criteria

**acknowledgments.md**:
- Added contributor recognition section
- Acknowledged OWASP community and project contributors

### Typo Fixes

**executive_summary/11_threat_modeling.md**:
- Fixed: "Theat Modeling" → "Threat Modeling"

---

## Statistics

### Content Additions
- **40+ new code examples** (C implementations for PQC, injection prevention, TEE, Yocto BitBake configs)
- **3 comprehensive PQC sections** (~500 lines of content)
- **4 comprehensive Yocto sections** (~1,520 lines of content across 4 chapters)
- **2 new appendix documents** (OWASP IoT Ecosystem, Compliance Frameworks)
- **9 compliance frameworks** documented
- **4 OWASP IoT projects** integrated
- **2 command injection examples** with vulnerable + secure code
- **1 comprehensive Yocto security checklist** (40 points)

### Files Modified
- **13 chapter files** updated with content
- **11 chapter files** updated with link fixes
- **8 files** updated with year references
- **2 files** created from empty state
- **2 new appendix documents** created

### Link Updates
- **36 OWASP links** migrated to new infrastructure
- **12+ CERT Secure Coding links** updated
- **7 new PQC resources** added
- **4 OWASP IoT project links** added

---

## OWASP IoT Ecosystem Mapping Summary

Every chapter now includes specific mappings:

| Chapter | ISVS Requirements | ISTG Test Cases | FSTM Stages | IoTGoat Labs |
|---------|-------------------|-----------------|-------------|--------------|
| 1. Buffer Overflow | V2.1.1, V2.1.2 | ISTG-FW-VULN-001 | Stage 5 | Buffer overflow |
| 2. Injection | V2.3.1, V2.3.2 | ISTG-UI-INFO-001 | Stage 4, 6 | Command injection |
| 3. Firmware Updates | V3.2.1-V3.2.3 | ISTG-FW-INFO-001 | Stage 3, 5 | Signature bypass |
| 4. Sensitive Info | V3.3.1, V3.3.2 | ISTG-FW-CRYPT-001 | Stage 4, 6 | Hardcoded keys |
| 5. Identity | V3.1.1, V3.1.2 | ISTG-FW-INFO-002 | Stage 2 | Device fingerprint |
| 8. TLS | V4.1.1, V4.2.1 | ISTG-DES-COMM-001 | Stage 6, 7 | TLS interception |
| 10. Third-Party | V1.2.1, V1.2.2 | ISTG-FW-INFO-003 | Stage 1 | SBOM generation |

*(Complete mapping available in [Appendix A](owasp-iot-ecosystem-alignment.md))*

---

## Post-Quantum Cryptography Timeline

### 2025-2026: Preparation & Testing
- Audit current cryptographic implementations
- Evaluate PQC libraries (liboqs, PQClean)
- Test hybrid signatures and encryption
- Update bootloaders for larger PQC signatures
- Deploy hybrid PQC in test environments

### 2026-2027: Hybrid Deployment
- Production rollout of hybrid classical+PQC
- Maintain backward compatibility
- Monitor performance impact
- Progressive re-encryption of stored data

### 2028+: PQC-Only
- Transition to PQC-only implementations
- Deprecate classical-only algorithms
- Full quantum-resistant protection

---

## Technology Versions Updated

### TLS/Cryptography
- **TLS 1.3** (RFC 8446) - Now recommended
- **TLS 1.2** - Minimum acceptable version
- **AES-256-GCM** - Recommended for quantum resistance
- **ChaCha20-Poly1305** - Modern AEAD cipher
- **ML-DSA (Dilithium)** - NIST FIPS 204 for signatures
- **ML-KEM (Kyber)** - NIST FIPS 203 for key encapsulation

### SBOM Tools
- **SPDX** - ISO/IEC 5962:2021
- **CycloneDX** - v1.6
- **Syft** - Modern SBOM generator
- **Trivy** - Comprehensive scanner
- **Grype** - Vulnerability matching
- **OSV-Scanner** - Open source vulnerabilities

### Hardware Security
- **ARM TrustZone** - TEE implementation
- **Intel SGX** - x86 secure enclaves
- **TPM 2.0** - Hardware root of trust
- **ATECC608** - I2C crypto coprocessor
- **EdgeLock SE050** - NXP secure element
- **OPTIGA Trust M** - Infineon automotive SE

---

## Resources Added

### Post-Quantum Cryptography
- [NIST PQC Project](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [FIPS 203 (ML-KEM)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf)
- [FIPS 204 (ML-DSA)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf)
- [liboqs - Open Quantum Safe](https://github.com/open-quantum-safe/liboqs)
- [OQS OpenSSL Provider](https://github.com/open-quantum-safe/oqs-provider)
- [wolfSSL PQC](https://www.wolfssl.com/post-quantum-cryptography/)
- [Cloudflare PQC Research](https://blog.cloudflare.com/post-quantum-for-all/)

### OWASP IoT Ecosystem
- [OWASP ISVS](https://owasp.org/www-project-internet-of-things-security-verification-standard/)
- [OWASP ISTG](https://github.com/OWASP/IoT-Security-Testing-Guide)
- [OWASP FSTM](https://github.com/OWASP/owasp-fstm)
- [OWASP IoTGoat](https://github.com/OWASP/IoTGoat)

### Compliance & Standards
- [EU Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act)
- [ETSI EN 303 645](https://www.etsi.org/deliver/etsi_en/303600_303699/303645/02.01.01_60/en_303645v020101p.pdf)
- [IEC 62443](https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards)
- [NIST IoT Cybersecurity](https://www.nist.gov/programs-projects/nist-cybersecurity-iot-program)
- [FDA Cybersecurity](https://www.fda.gov/medical-devices/digital-health-center-excellence/cybersecurity)

---

## Breaking Changes

### None

All updates are backward compatible. The guide maintains support for existing implementations while recommending modern standards for new projects.

---

## Deprecation Notices

### Cryptography
- **TLS 1.0/1.1**: Deprecated, do not use
- **SSLv3**: Deprecated, do not use
- **MD5**: Deprecated for signatures
- **SHA-1**: Deprecated for signatures
- **RSA-1024**: Deprecated, minimum RSA-2048
- **DES/3DES**: Deprecated, use AES

### Recommendations for Migration
- TLS 1.0/1.1 → TLS 1.2/1.3 by Q2 2026
- Classical-only crypto → Hybrid PQC by 2027
- RSA/ECDSA signatures → Consider adding ML-DSA for long-term protection

---

## Future Roadmap

### Planned for 2026
- Zero Trust Architecture for embedded systems
- eBPF for runtime security monitoring
- Rust memory safety examples
- Supply chain attack case studies
- AI/ML security considerations for edge devices

### Monitoring
- NIST PQC standardization updates
- EU Cyber Resilience Act implementation
- TLS 1.4 standardization
- OWASP IoT project developments

---

## Contributors

This modernization was made possible by:
- OWASP Embedded Application Security Project Team
- OWASP IoT Security Project Contributors
- NIST Post-Quantum Cryptography Project
- Open Quantum Safe (OQS) Project
- Community feedback and issue reporters

---

## Verification

To verify the updates:

```bash
# Check OWASP IoT ecosystem references
grep -r "OWASP ISVS\|OWASP ISTG\|OWASP FSTM\|IoTGoat" [1-9]*.md

# Check PQC content
grep -r "ML-DSA\|ML-KEM\|post-quantum" [3,4,8]*.md

# Check TLS 1.3 references
grep -r "TLS 1.3\|TLS_AES_256_GCM" 8*.md

# Check compliance frameworks
cat device-compliance-frameworks.md | grep -E "EU Cyber|ETSI|IEC 62443"
```

---

**Changelog Version**: 2025.1
