# OWASP Embedded Application Security - Release Notes v2025.1

**Release Date**: October 2025
**Version**: 2025.1
**Repository**: https://github.com/scriptingxss/embeddedappsec

---

## Overview

This is a major modernization release of the OWASP Embedded Application Security Best Practices guide. The project has been comprehensively updated to reflect current security standards, emerging threats (quantum computing, AI/ML), and modern compliance requirements for embedded and IoT devices.

---

## Key Highlights

### 🔐 Post-Quantum Cryptography Readiness
- Comprehensive PQC guidance across Chapters 3, 4, and 8
- NIST PQC standards (ML-DSA, ML-KEM, SLH-DSA)
- Hybrid cryptography implementations
- Migration roadmaps for devices with 10-20+ year lifecycles

### 🏗️ Yocto Project Security Integration
- 780+ lines of Yocto build system security guidance
- Complete BitBake configurations for secure embedded Linux
- Hardware security integration (OP-TEE, TPM 2.0, Secure Elements)
- CVE checking and SBOM generation workflows

### 📋 Compliance Framework Alignment
- EU Cyber Resilience Act
- IEC 62443 (Industrial IoT)
- ETSI EN 303 645 (Consumer IoT)
- ISO/SAE 21434 (Automotive)
- NIST IoT Cybersecurity
- FDA Medical Device Cybersecurity

### 🔗 OWASP IoT Ecosystem Integration
- Cross-references to OWASP ISVS, ISTG, FSTM, IoTGoat
- Testable security requirements
- Hands-on training resources

### 📚 Content Expansion
- +4,000 lines of new security guidance
- Production-ready code examples (C, BitBake recipes)
- Real-world vulnerability case studies
- Automated security workflows

---

## What's New

### Major Additions
- **Chapter 1**: Compiler hardening (+560 lines)
- **Chapter 2**: Injection prevention examples (+698 lines)
- **Chapter 3**: Firmware updates & PQC (+992 lines)
- **Chapter 6**: Platform security restructuring
- **Chapter 8**: TLS & PQC (+865 lines)
- **Chapter 10**: SBOM & CVE management (+797 lines)
- **Threat Model**: Manifesto alignment and anti-patterns
- **Appendix A**: OWASP IoT ecosystem alignment
- **Appendix B**: Device compliance frameworks

### Content Improvements
- Removed duplicate content across chapters
- Added cross-references for better navigation
- Updated all OWASP wiki links
- Removed year-date labels from headers for evergreen docs
- Fixed broken external references

---

## Compliance & Standards Coverage

This release addresses requirements from:
- EU Cyber Resilience Act (CRA) - SBOM mandates, security updates
- Executive Order 14028 - Software supply chain security
- NTIA Minimum Elements for SBOM
- PCI DSS 4.0 - TLS requirements
- NIST SP 800-52 Rev. 2 - TLS guidelines
- FIPS 140-2/140-3 - Cryptographic module validation

---

## Target Audiences

- **IoT Device Manufacturers** - Consumer devices, smart home, edge computing
- **Industrial IoT Developers** - SCADA, ICS, factory automation, Industry 4.0
- **Automotive Engineers** - Connected vehicles, ECUs, ADAS systems
- **Medical Device Developers** - FDA-regulated connected medical equipment
- **Enterprise Network Equipment Vendors** - Routers, switches, firewalls, SD-WAN

---

## Breaking Changes

**None** - This is a content-additive release. All existing guidance remains valid.

---

## Known Issues

Open GitHub issues for broken external links (issues #12-#23) are being addressed. Most referenced content has been replaced with current alternatives.

---

## How to Use This Release

### For Developers
1. Start with [preface.md](preface.md) for scope and definitions
2. Review [threat-model.md](threat-model.md) for threat modeling frameworks
3. Follow chapter-specific guidance for your platform (Yocto, Buildroot, RTOS)
4. Use [device-compliance-frameworks.md](device-compliance-frameworks.md) for regulatory requirements

### For Security Teams
1. Review [owasp-iot-ecosystem-alignment.md](owasp-iot-ecosystem-alignment.md) for testing methodologies
2. Map requirements to OWASP ISVS verification standards
3. Use Chapter 10 for supply chain security and SBOM generation
4. Implement CVE scanning workflows from Chapter 10

### For Management
1. Check [device-compliance-frameworks.md](device-compliance-frameworks.md) for market requirements
2. Review [project-roadmap.md](project-roadmap.md) for ongoing developments
3. Use this guide for security training and onboarding

---

## Contributors

Special thanks to all contributors who made this modernization possible. See [acknowledgments.md](acknowledgments.md) for full list.

---

## Next Steps

See [CHANGELOG-2025.md](CHANGELOG-2025.md) for detailed technical changes.

**Project Leaders**:
- Aaron Guzman [@scriptingxss](https://twitter.com/scriptingxss)
- Alex Lafrenz [@zerofrenz](https://twitter.com/zerofrenz)

**Project Page**: https://owasp.org/www-project-embedded-application-security/
