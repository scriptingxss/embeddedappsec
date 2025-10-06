# Modern Compliance Frameworks for Embedded and IoT Security (2025)

This document provides guidance on aligning embedded device security with current regulatory and industry compliance frameworks.

## Table of Contents

1. [EU Cyber Resilience Act (CRA)](#eu-cyber-resilience-act-cra)
2. [ETSI EN 303 645](#etsi-en-303-645)
3. [IEC 62443](#iec-62443)
4. [NIST IoT Cybersecurity](#nist-iot-cybersecurity)
5. [FDA Medical Device Cybersecurity](#fda-medical-device-cybersecurity)
6. [Automotive (ISO/SAE 21434, UN R155/R156)](#automotive-standards)
7. [US Executive Order 14028](#us-executive-order-14028)
8. [California IoT Security Law (SB-327)](#california-iot-security-law)
9. [Compliance Mapping](#compliance-mapping)

---

## EU Cyber Resilience Act (CRA)

**Status**: Approved 2025, phased implementation through 2027
**Scope**: Products with digital elements sold in EU market
**Authority**: European Commission

### Key Requirements

1. **Secure by Design**:
   * Security integrated throughout product lifecycle
   * Risk-based approach to cybersecurity
   * Documented security risk assessments

2. **Vulnerability Management**:
   * Handle vulnerabilities throughout product lifetime
   * Provide security updates for minimum 5 years (or expected lifetime)
   * Report actively exploited vulnerabilities within 24 hours to ENISA

3. **Software Bill of Materials (SBOM)**:
   * Maintain comprehensive SBOM
   * Make available to authorities upon request
   * Track all components and dependencies

4. **Essential Cybersecurity Requirements** (Annex I):
   * No known exploitable vulnerabilities
   * Secure by default configuration
   * Protection against unauthorized access
   * Minimize attack surface
   * Secure software updates with rollback
   * Log security-relevant events
   * Ensure data confidentiality and integrity

5. **Conformity Assessment**:
   * Critical products (Annex III): Third-party assessment
   * Other products: Self-assessment with documentation

### Implementation for Embedded Devices

**Required Documentation**:
* EU Declaration of Conformity
* Technical documentation
* Security risk assessment
* SBOM
* Instructions for secure use

**Best Practice References**:
* Chapter 3: Firmware Updates → Aligns with CRA secure update requirements
* Chapter 4: Securing Sensitive Information → Supports data protection requirements
* Chapter 10: Third Party Code → SBOM and component management

### Resources

* [EU Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act)
* [CRA Full Text](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:52022PC0454)

---

## ETSI EN 303 645

**Title**: Cyber Security for Consumer Internet of Things
**Version**: V2.1.1 (2020-06)
**Scope**: Consumer IoT devices
**Status**: Referenced by UK and EU regulations

### 13 Key Provisions

1. **No universal default passwords**
2. **Implement vulnerability disclosure policy**
3. **Keep software updated**
4. **Securely store sensitive security parameters**
5. **Communicate securely**
6. **Minimize exposed attack surfaces**
7. **Ensure software integrity**
8. **Ensure personal data is secure**
9. **Make systems resilient to outages**
10. **Examine system telemetry data**
11. **Make it easy for users to delete user data**
12. **Make installation and maintenance easy**
13. **Validate input data**

### Mapping to Best Practices

| ETSI Provision | Guide Chapter | Implementation |
|----------------|---------------|----------------|
| Provision 1 | Chapter 5 (Identity Management) | Unique passwords per device, password complexity |
| Provision 2 | Chapter 7 (Debugging Code) | Vulnerability disclosure process |
| Provision 3 | Chapter 3 (Firmware Updates) | Secure OTA updates, automatic updates |
| Provision 4 | Chapter 4 (Securing Sensitive Info) | Hardware security (TEE/SE), encryption |
| Provision 5 | Chapter 8 (TLS) | TLS 1.3, certificate validation |
| Provision 6 | Chapter 6 (Framework Hardening) | Minimize services, disable debug interfaces |
| Provision 7 | Chapter 3 (Firmware Updates) | Cryptographic signatures, secure boot |
| Provision 8 | Chapter 9 (Privacy) | Data minimization, secure storage |
| Provision 13 | Chapter 2 (Injection Prevention) | Input validation, sanitization |

### Resources

* [ETSI EN 303 645 Specification](https://www.etsi.org/deliver/etsi_en/303600_303699/303645/)
* [Implementation Guide](https://www.etsi.org/images/files/ETSIWhitePapers/etsi_wp31_CYBER_Implementation_Guide.pdf)

---

## IEC 62443

**Title**: Industrial Communication Networks - IT Security for Networks and Systems
**Scope**: Industrial Automation and Control Systems (IACS)
**Parts**: 4-1 (Secure Product Development), 4-2 (Technical Security Requirements)

### Security Levels (SL)

* **SL 1**: Protection against casual or coincidental violation
* **SL 2**: Protection against intentional violation using simple means
* **SL 3**: Protection against intentional violation using sophisticated means
* **SL 4**: Protection against intentional violation using sophisticated means with extended resources

### Key Requirements

**IEC 62443-4-1 (Secure Development Lifecycle)**:
1. Security management
2. Specification of security requirements
3. Secure by design
4. Secure implementation
5. Security verification and validation
6. Security update management
7. Security guidelines

**IEC 62443-4-2 (Technical Requirements)**:
* Foundational Requirements (FR):
  * FR 1: Identification and Authentication Control
  * FR 2: Use Control
  * FR 3: System Integrity
  * FR 4: Data Confidentiality
  * FR 5: Restricted Data Flow
  * FR 6: Timely Response to Events
  * FR 7: Resource Availability

### Implementation Mapping

| IEC 62443 FR | Guide Chapter | Key Controls |
|--------------|---------------|--------------|
| FR 1 | Chapter 5 | Multi-factor authentication, unique device identity |
| FR 2 | Chapter 5 | Role-based access control, least privilege |
| FR 3 | Chapter 3, 6 | Secure boot, integrity verification, input validation |
| FR 4 | Chapter 4, 8 | Encryption at rest/transit, key management |
| FR 5 | Chapter 6 | Network segmentation, firewall rules |
| FR 6 | Chapter 6 | Logging, monitoring, incident response |
| FR 7 | Chapter 6 | DoS protection, resource management |

### Resources

* [ISA/IEC 62443 Standards](https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards)
* [IEC 62443-4-1](https://webstore.iec.ch/publication/33615)
* [IEC 62443-4-2](https://webstore.iec.ch/publication/34421)

---

## NIST IoT Cybersecurity

**Title**: NIST Cybersecurity for IoT Program
**Key Publications**:
* NISTIR 8259 series
* NIST SP 800-213 (IoT Device Cybersecurity Guidance)

### Core Baseline Capabilities (NISTIR 8259A)

1. **Device Identification**
2. **Device Configuration**
3. **Data Protection**
4. **Logical Access to Interfaces**
5. **Software/Firmware Update**
6. **Cybersecurity State Awareness**

### NIST SP 800-213 Recommendations

**Before Purchase**:
* Review manufacturer's cybersecurity capabilities
* Evaluate update/support lifecycle
* Assess data handling practices

**During Integration**:
* Change default credentials
* Disable unnecessary services
* Configure secure communications
* Implement network segmentation

**Operational Phase**:
* Monitor for anomalies
* Apply updates promptly
* Review logs regularly
* Maintain asset inventory

### Resources

* [NIST IoT Cybersecurity](https://www.nist.gov/itl/applied-cybersecurity/nist-cybersecurity-iot-program)
* [NISTIR 8259A](https://nvlpubs.nist.gov/nistpubs/ir/2020/NIST.IR.8259A.pdf)
* [NIST SP 800-213](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-213.pdf)

---

## FDA Medical Device Cybersecurity

**Authority**: US Food and Drug Administration
**Guidance**: Cybersecurity in Medical Devices (2023)
**Scope**: Medical devices with software/connectivity

### Key Requirements

1. **Premarket Submissions**:
   * Cybersecurity Bill of Materials (CBOM/SBOM)
   * Threat modeling and risk analysis
   * Security architecture and design
   * Software verification and validation
   * Instructions for secure use

2. **Post-Market**:
   * Vulnerability management plan
   * Coordinated vulnerability disclosure
   * Security update deployment
   * End-of-support planning

3. **Specific Controls**:
   * Authentication and authorization
   * Encryption (data at rest and in transit)
   * Secure communications (TLS 1.2+)
   * Audit logging
   * Physical tamper protection
   * Secure software updates

### FDA Cybersecurity Tiers

* **Tier 1**: Minimal cybersecurity concerns
* **Tier 2**: Moderate cybersecurity concerns
* **Tier 3**: Major cybersecurity concerns (life-sustaining devices)

Higher tiers require more rigorous security controls and documentation.

### Resources

* [FDA Medical Device Cybersecurity](https://www.fda.gov/medical-devices/digital-health-center-excellence/cybersecurity)
* [Premarket Guidance (2023)](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/cybersecurity-medical-devices-quality-system-considerations-and-content-premarket-submissions)

---

## Automotive Standards

### ISO/SAE 21434 - Road Vehicles Cybersecurity Engineering

**Published**: 2021
**Scope**: Cybersecurity for road vehicles throughout lifecycle

**Key Phases**:
1. Concept Phase
2. Product Development
3. Production
4. Operations and Maintenance
5. End of Cybersecurity Support

**Security Requirements**:
* Threat Analysis and Risk Assessment (TARA)
* Cybersecurity by design
* Secure development lifecycle
* Vulnerability management
* Incident response

### UN R155/R156 (UNECE Regulations)

**R155**: Cybersecurity Management System (mandatory 2025)
**R156**: Software Update Management System (mandatory 2025)

**R155 Requirements**:
* Cybersecurity management system
* Risk assessment methodology
* Secure vehicle design
* Testing and verification
* Post-production monitoring

**R156 Requirements**:
* Secure software update processes
* Verification of authenticity
* Protection against unauthorized updates
* User notification procedures

### Automotive-Specific Guidance

| Standard Requirement | Guide Chapter | Implementation |
|---------------------|---------------|----------------|
| Secure Boot | Chapter 3, 6 | U-Boot hardening, verified boot |
| OTA Updates | Chapter 3 | Uptane, secure update protocols |
| V2X Security | Chapter 8 | TLS 1.3, certificate management |
| ECU Hardening | Chapter 6 | Toolchain hardening, service reduction |
| Key Management | Chapter 4 | HSM integration, TEE usage |

### Resources

* [ISO/SAE 21434](https://www.iso.org/standard/70918.html)
* [UN R155](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-155-cyber-security-and-cyber)
* [UN R156](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software)
* [Uptane for Automotive OTA](https://uptane.github.io/)

---

## US Executive Order 14028

**Title**: Improving the Nation's Cybersecurity
**Date**: May 2021
**Scope**: Federal software procurement and critical infrastructure

### Key Mandates

1. **Zero Trust Architecture**
2. **Supply Chain Security**:
   * SBOM required for all software
   * SLSA compliance recommended
   * Software provenance tracking

3. **Enhanced Logging**:
   * Comprehensive security event logging
   * Centralized log aggregation
   * Minimum retention periods

4. **Vulnerability Disclosure**:
   * Coordinated vulnerability disclosure (CVD)
   * Security.txt implementation
   * Transparency in handling vulnerabilities

### SBOM Requirements (Per NTIA Guidelines)

**Minimum Elements**:
* Supplier name
* Component name
* Version of component
* Other unique identifiers
* Dependency relationships
* Author of SBOM data

**Format**: SPDX or CycloneDX

### Resources

* [Executive Order 14028](https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/)
* [NTIA SBOM Minimum Elements](https://www.ntia.gov/files/ntia/publications/sbom_minimum_elements_report.pdf)
* [CISA Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model)

---

## California IoT Security Law (SB-327)

**Effective**: January 2020
**Scope**: IoT devices sold in California
**Requirement**: Reasonable security features

### Key Provisions

1. **Unique Passwords**:
   * No default passwords shared across devices
   * Either preprogrammed unique password per device
   * Or force password change on first use

2. **Reasonable Security**:
   * Appropriate to nature and function of device
   * Appropriate to information it may collect, contain, or transmit
   * Protection against unauthorized access, destruction, use, modification, or disclosure

### Best Practice Interpretation

The law is intentionally broad, but "reasonable security" should include:
* Secure authentication (Chapter 5)
* Data encryption (Chapter 4, 8)
* Secure updates (Chapter 3)
* Privacy protections (Chapter 9)
* Input validation (Chapter 2)

### Resources

* [SB-327 Full Text](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=201720180SB327)
* [California IoT Security Law Guide](https://oag.ca.gov/privacy/ccpa)

---

## Compliance Mapping

### Quick Reference Table

| Regulation | Geographic Scope | Device Type | Key Requirements | Related Chapters |
|------------|-----------------|-------------|------------------|------------------|
| **EU CRA** | European Union | All products with digital elements | SBOM, vuln mgmt, secure updates, 5-yr support | 3, 4, 10 |
| **ETSI EN 303 645** | EU/UK | Consumer IoT | 13 provisions, no default passwords, secure comms | 2, 3, 5, 6, 8 |
| **IEC 62443** | Global | Industrial/OT | Security levels, secure development, technical controls | All chapters |
| **NIST IoT** | US Federal | IoT devices | 6 core capabilities, device identity, updates | 3, 4, 5, 6 |
| **FDA** | United States | Medical devices | SBOM, threat model, vuln disclosure, security tiers | 3, 4, 10, 11 |
| **ISO/SAE 21434** | Automotive | Vehicles/ECUs | TARA, secure SDLC, post-production monitoring | All chapters |
| **UN R155/R156** | Europe/Japan+ | Vehicles | Cybersecurity mgmt system, secure OTA | 3, 6 |
| **EO 14028** | US Federal | Software procurement | SBOM, zero trust, enhanced logging | 6, 10 |
| **CA SB-327** | California | Consumer IoT | Unique passwords, reasonable security | 5, all |

### Implementation Priority by Market

**Targeting EU Market**:
1. EU Cyber Resilience Act compliance (top priority)
2. ETSI EN 303 645 alignment
3. SBOM generation (SPDX/CycloneDX)
4. 5-year security support commitment

**Targeting US Federal**:
1. NIST IoT guidance implementation
2. SBOM with NTIA minimum elements
3. SLSA framework adoption
4. FedRAMP considerations (if cloud-connected)

**Automotive Industry**:
1. ISO/SAE 21434 compliance
2. UN R155/R156 certification
3. Uptane OTA implementation
4. AUTOSAR security modules

**Medical Devices**:
1. FDA premarket cybersecurity submission
2. SBOM for all components
3. Vulnerability management plan
4. Coordinated disclosure program

**General Consumer IoT**:
1. ETSI EN 303 645 (UK/EU)
2. CA SB-327 (if selling in California)
3. NIST guidance as best practice
4. Industry-specific standards (if applicable)

---

## Compliance Checklist

### Pre-Development

- [ ] Identify applicable regulations based on target markets
- [ ] Perform threat modeling (Chapter 11)
- [ ] Document security requirements
- [ ] Establish secure development lifecycle
- [ ] Set up SBOM generation toolchain (Chapter 10)

### During Development

- [ ] Implement security controls per compliance requirements
- [ ] Follow best practices from this guide (Chapters 1-10)
- [ ] Conduct security testing (leverage OWASP ISTG)
- [ ] Generate and maintain SBOM
- [ ] Document security architecture

### Pre-Release

- [ ] Security verification and validation
- [ ] Penetration testing (use OWASP FSTM methodology)
- [ ] Vulnerability assessment
- [ ] Create security documentation for users
- [ ] Prepare compliance declarations

### Post-Release

- [ ] Vulnerability disclosure process
- [ ] Security update deployment
- [ ] Incident response procedures
- [ ] Continuous monitoring
- [ ] Regular security assessments

---

## Additional Resources

### Compliance Tools and Frameworks

* [OpenChain](https://www.openchainproject.org/) - Open source compliance
* [SPDX](https://spdx.dev/) - SBOM standard
* [CycloneDX](https://cyclonedx.org/) - SBOM for security
* [SLSA](https://slsa.dev/) - Supply chain security
* [Sigstore](https://www.sigstore.dev/) - Software signing

### Industry Organizations

* [IIC (Industrial Internet Consortium)](https://www.iiconsortium.org/) - Industrial IoT security
* [OCF (Open Connectivity Foundation)](https://openconnectivity.org/) - IoT standards
* [GSMA IoT Security](https://www.gsma.com/iot/iot-security/) - Mobile IoT security
* [CSA (Connectivity Standards Alliance)](https://csa-iot.org/) - Matter protocol

### Government Resources

* [CISA IoT Security](https://www.cisa.gov/topics/critical-infrastructure-security-and-resilience/securing-internet-things)
* [ENISA IoT Security](https://www.enisa.europa.eu/topics/iot-and-smart-infrastructures)
* [BSI (German Federal Office)](https://www.bsi.bund.de/EN/Home/home_node.html) - IoT security guidelines

