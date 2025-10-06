# What are Embedded Systems?

The term embedded or embedded systems can be interpreted in several ways depending on your background, knowledge, and exposure to embedded technology. For the purpose of this document, **firmware** is defined as the software layer between the underlying hardware and the operating system (OS). The main purpose of firmware is to initialize and abstract enough hardware so operating systems drivers and components can further configure the hardware according to its functionality.

In 2025, embedded systems span from 8-bit microcontrollers in IoT sensors to multi-core ARM processors running Linux in automotive systems, industrial equipment, and smart infrastructure. This guide addresses security considerations across this entire spectrum.

## Authoritative Standards and Definitions (2025)

This guide aligns with internationally recognized standards and frameworks:

### Security Standards

* **[IEC 62443](https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards)** - Industrial Automation and Control Systems Security
  * IEC 62443-4-1: Secure Product Development Lifecycle
  * IEC 62443-4-2: Technical Security Requirements for Components
* **[ETSI EN 303 645](https://www.etsi.org/deliver/etsi_en/303600_303699/303645/)** - Cyber Security for Consumer Internet of Things (Baseline Requirements)
* **[ISO/SAE 21434:2021](https://www.iso.org/standard/70918.html)** - Road Vehicles — Cybersecurity Engineering
* **[NIST SP 800-160 Vol. 2](https://csrc.nist.gov/publications/detail/sp/800-160/vol-2/final)** - Developing Cyber-Resilient Systems: A Systems Security Engineering Approach
* **[IEEE 802.1AR-2018](https://standards.ieee.org/standard/802_1AR-2018.html)** - Secure Device Identity

### Device Identity and Attestation

* **[TCG DICE](https://trustedcomputinggroup.org/work-groups/dice-architectures/)** - Device Identifier Composition Engine
* **[TCG TPM 2.0](https://trustedcomputinggroup.org/resource/tpm-library-specification/)** - Trusted Platform Module Library Specification
* **[ARM PSA Certified](https://www.psacertified.org/)** - Platform Security Architecture for IoT Devices

### Regulatory Frameworks

* **[EU Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act)** - Horizontal cybersecurity requirements for products with digital elements (2025+)
* **[UNECE WP.29 R155/R156](https://unece.org/transport/vehicle-regulations-wp29)** - Cybersecurity and Software Update Management Systems for Vehicles
* **[FDA Cybersecurity in Medical Devices](https://www.fda.gov/medical-devices/digital-health-center-excellence/cybersecurity)** - Premarket and postmarket cybersecurity guidance

### Supply Chain Security

* **[NTIA Software Bill of Materials (SBOM)](https://www.ntia.gov/sbom)** - Minimum Elements for a Software Bill of Materials
* **[SPDX (ISO/IEC 5962:2021)](https://spdx.dev/)** - Software Package Data Exchange standard
* **[CycloneDX](https://cyclonedx.org/)** - OWASP Software Bill of Materials standard for security use cases
* **[SLSA Framework](https://slsa.dev/)** - Supply-chain Levels for Software Artifacts

---

## Hardware

### Resource Spectrum (2025)

* **Tiny IoT** (Constrained devices - IETF RFC 7228 Class 1/2):
  * 16KB - 256KB RAM
  * 32KB - 2MB Flash storage
  * Examples: nRF52840, ESP32-C3, STM32L0
* **Standard Embedded** (Class 2):
  * 256KB - 16MB RAM
  * 2MB - 128MB Flash storage
  * Examples: STM32F7, i.MX RT1060, ESP32-S3
* **High-Performance Embedded** (Embedded Linux capable):
  * 64MB - 2GB+ RAM
  * 256MB - 32GB Flash/eMMC storage
  * Examples: i.MX 8M, Raspberry Pi, BeagleBone

### Form Factors

* **System-on-Chip (SoC)**: Complete computer system integrated into single chip
* **System-on-Module (SoM)**: Pre-integrated module with SoC, RAM, flash, PMU
* **Microcontroller (MCU)**: Single-chip computer for specific control applications
* **Secure Elements**: Dedicated cryptographic co-processors (ATECC608, SE050, TPM 2.0)

## Bootloaders

* **U-Boot** (Das U-Boot) - Universal Boot Loader, most popular for ARM/embedded Linux
* **Grub2** - Grand Unified Bootloader, x86/x64 systems
* **Coreboot** - Lightweight firmware replacement (formerly LinuxBIOS)
* **EDK II** - UEFI reference implementation
* **Little Kernel (LK)** - Used in Android bootloader chain
* **Barebox** - U-Boot fork with enhanced devicetree support
* **TF-A** - Trusted Firmware-A (ARM Trusted Firmware)
* **MCUboot** - Secure bootloader for 32-bit MCUs (Zephyr, Mbed OS)

## CPU Architectures (2025)

### Dominant Architectures

* **ARM** (Dominant in embedded, IoT, automotive, mobile)
  * Cortex-M series (M0+, M4, M33, M55, M85) - Microcontrollers with TrustZone-M
  * Cortex-A series (A53, A72, A78) - Application processors for embedded Linux
  * Cortex-R series (R5, R52) - Real-time safety-critical systems
* **RISC-V** (Emerging open-source architecture gaining rapid adoption)
  * E/I series - Embedded applications
  * SiFive U series - Linux-capable cores
  * Examples: ESP32-C3/C6, StarFive VisionFive, SiFive HiFive
* **x86/x64** (Intel/AMD) - Industrial PCs, gateways, edge computing
* **MIPS** (Legacy, declining) - Router/networking equipment
* **PowerPC** (NXP QorIQ) - Industrial automation, automotive ECUs
* **AVR** (Microchip) - 8-bit microcontrollers (Arduino)
* **Xtensa** (Cadence/Espressif) - ESP32, ESP8266 Wi-Fi SoCs

### Word Lengths

* **8-bit**: AVR, PIC, legacy systems
* **16-bit**: MSP430, some PIC variants
* **32-bit**: ARM Cortex-M, RISC-V RV32, most modern MCUs
* **64-bit**: ARM Cortex-A (AArch64), RISC-V RV64, x86-64

## Operating System Platforms (2025)

### Embedded Linux Distributions

* **Yocto/OpenEmbedded** - Custom Linux distribution builder (industry standard)
* **Buildroot** - Lightweight build system for embedded Linux
* **OpenWrt** - Linux for routers and networking equipment
* **Ubuntu Core** - Snaps-based minimal Ubuntu for IoT
* **Raspberry Pi OS** - Debian-based for Raspberry Pi
* **Android AOSP** - Android Open Source Project for embedded devices

### Real-Time Operating Systems (RTOS)

* **FreeRTOS** - Most popular open-source RTOS (Amazon)
* **Zephyr RTOS** - Linux Foundation, highly scalable, security-focused
* **Azure RTOS (ThreadX)** - Microsoft embedded OS (now open-source)
* **Mbed OS** - ARM's IoT OS with PSA security
* **QNX** - Commercial microkernel RTOS (automotive, medical)
* **VxWorks** - Commercial RTOS (aerospace, industrial)
* **AUTOSAR** - Automotive software architecture standard
* **INTEGRITY** - Green Hills safety-critical RTOS (DO-178C certified)
* **NuttX** - POSIX-compliant RTOS
* **RIOT** - IoT-focused RTOS with network stack

### Bare-Metal / No-OS

* **Bare-metal C/C++** - Direct hardware programming without OS
* **Arduino framework** - Simplified bare-metal for hobbyists/education
* **STM32 HAL/LL** - ST Microelectronics hardware abstraction
* **ESP-IDF** - Espressif IoT Development Framework

### Legacy/Declining Platforms

* **Windows Embedded Compact** (Windows CE) - End-of-life
* **Windows 10 IoT** - Replaced by Windows 11 IoT Enterprise

## Programming Languages (2025)

### Systems Programming (Firmware/Drivers)

* **C** - Dominant language for embedded systems, kernel, drivers
* **C++** - Modern C++ (C++11/14/17/20) for embedded applications
* **Rust** - Memory-safe systems programming (gaining rapid adoption)
  * Used in: Zephyr RTOS, Android, Linux kernel modules
  * Embedded frameworks: embedded-hal, RTIC, Embassy
* **Assembly** - Architecture-specific optimizations, bootloader code
* **Zig** - Emerging C alternative with compile-time safety

### Application Layer / Scripting

* **Python** (MicroPython, CircuitPython) - Rapid prototyping, test scripts
* **JavaScript/TypeScript** (Node.js, Embedded JavaScript) - IoT gateways
* **Go (Golang)** - Cloud connectivity, backend services on gateways
* **Lua** - Lightweight scripting embedded in firmware
* **Elixir (Nerves)** - Functional programming for IoT

### Web Interfaces (Embedded HTTP Servers)

* **C/C++** - libmicrohttpd, Mongoose, lighttpd
* **Go** - net/http for resource-rich devices
* **Rust** - actix-web, warp for secure web services
* **PHP** - Legacy devices (security concerns, avoid for new projects)
* **Classic ASP** - Obsolete (end-of-life, security risk)

## Product Lifespan and Support Requirements (2025)

Embedded device lifespans vary by industry, with security support obligations increasingly mandated by regulation:

### Industry-Specific Lifespans

* **Consumer IoT**: 2-5 years (ETSI EN 303 645: minimum defined support period)
* **Smart Home**: 5-10 years (Matter/Thread devices)
* **Automotive**: 10-15 years (UNECE WP.29 requires cybersecurity support over vehicle lifetime)
* **Medical Devices**: 7-15 years (FDA cybersecurity postmarket guidance)
* **Industrial IoT/ICS**: 15-25+ years (IEC 62443 lifecycle requirements)
* **Critical Infrastructure**: 20-30+ years ("immortal" systems)
* **Aerospace/Defense**: 30-50+ years

### Regulatory Support Requirements

* **EU Cyber Resilience Act**: Minimum 5 years security updates (or expected product lifetime)
* **ETSI EN 303 645**: Defined support period disclosed to consumers
* **California IoT Law (SB-327)**: Reasonable security features for device lifetime
* **UNECE WP.29 R156**: Software update capability throughout vehicle life

### Flash Memory Considerations

* NAND Flash: ~10,000-100,000 write cycles (with wear leveling)
* NOR Flash: ~100,000-1,000,000 write cycles
* eMMC/UFS: Similar to NAND with controller-managed wear leveling
* Data retention: 10-20 years specified by manufacturers (temperature dependent)

---

## Modern Security Considerations (2025)

### Hardware Security Features

* **Trusted Execution Environment (TEE)**: ARM TrustZone, Intel SGX, RISC-V MultiZone
* **Secure Boot**: ROM-based root of trust, verified boot chains (U-Boot Verified Boot, UEFI Secure Boot)
* **Hardware Crypto Accelerators**: AES-NI, SHA extensions, ECC/RSA accelerators
* **Secure Elements**: TPM 2.0, Discrete secure elements (ATECC608, EdgeLock SE050)
* **Physical Unclonable Functions (PUF)**: Device-unique hardware identities
* **Post-Quantum Crypto**: Emerging support for NIST PQC algorithms (Kyber, Dilithium)

### Software Security Trends

* **Memory-Safe Languages**: Rust adoption in Linux kernel, Zephyr, Android
* **Formal Verification**: Critical code paths mathematically proven correct
* **Software Bill of Materials (SBOM)**: Mandatory for supply chain transparency
* **Zero Trust Architecture**: Device identity, attestation, conditional access
* **Secure OTA Updates**: Encrypted, signed firmware with rollback protection

