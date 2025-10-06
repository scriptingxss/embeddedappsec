# Third Party Code and Components

Following setup of the toolchain, it is important to ensure that the kernel, software packages, and third party libraries are updated to protect against publicly known vulnerabilities. Software such as Rompager or embedded build tools such as Buildroot should be checked against vulnerability databases as well as their ChangeLogs to determine when and if an update is needed. It is important to note this process should be tested by developers and/or QA teams prior to release builds as updates to embedded systems can cause issues with the operations of those systems.

Embedded projects should maintain a “Bill of Materials” of the third party and open source software included in its firmware images. This Bill of Materials should be checked to confirm that none of the third party software included has any unpatched vulnerabilities and also. Up to date vulnerability information may be found through the National Vulnerability Database or Open Hub.

Several solutions exist for cataloging and auditing third party software. Many solutions are built into your build environment such as:

* C / C++
  * `Makefile`
* Go
  * Use the official `dep` [tool](https://github.com/golang/dep)
* Node
  * `npm list`
* Python
  * `pip freeze`
* Ruby
  * `gem dependency`
* Lua
  * See the `rockspec file`
* Java
  * `mvn dependency:tree`
  * `gradle app:dependencies`
* Yocto
  * `buildhistory`
* Buildroot (free)
  * `make legal-info`
* Package Managers (free)
*
  * `dpkg --list`
  * `rpm -qa`
  * `yum list`
  * `apt list --installed`
* RetireJS for Javascript projects (free)

**A sample BOM is shown below:**

| **Component** | Version | Vulnerabilities - CVEs | Notes       |
| ------------- | ------- | ---------------------- | ----------- |
| jQuery        | 1.4.4   | CVE-2011-4969          |             |
| libxml2       | 2.9.4   | CVE-2016-5131          | To be fixed |

Software BOM's also include licensing and contextual information relating to the function of the component or justification for using the specific version.

**Retirejs in a JavaScript project directory Example:**

```bash
$ retire .
Loading from cache: https://raw.githubusercontent.com/RetireJS/retire.js/master/repository/jsrepository.json
Loading from cache: https://raw.githubusercontent.com/RetireJS/retire.js/master/repository/npmrepository.json
/js/jquery-1.4.4.min.js
 ↳ jquery 1.4.4.min has known vulnerabilities: severity: medium; CVE: CVE-2011-4969; http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2011-4969 http://research.insecurelabs.org/jquery/test/ severity: medium; bug: 11290, summary: Selector interpreted as HTML; http://bugs.jquery.com/ticket/11290 http://research.insecurelabs.org/jquery/test/ severity: medium; issue: 2432, summary: 3rd party CORS request may execute; https://github.com/jquery/jquery/issues/2432 http://blog.jquery.com/2016/01/08/jquery-2-2-and-1-12-released/
/js/jquery-1.4.4.min.js
 ↳ jquery 1.4.4.min has known vulnerabilities: severity: medium; CVE: CVE-2011-4969; http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2011-4969 http://research.insecurelabs.org/jquery/test/ severity: medium; bug: 11290, summary: Selector interpreted as HTML; http://bugs.jquery.com/ticket/11290 http://research.insecurelabs.org/jquery/test/ severity: medium; issue: 2432, summary: 3rd party CORS request may execute; https://github.com/jquery/jquery/issues/2432 http://blog.jquery.com/2016/01/08/jquery-2-2-and-1-12-released/
/javascript/vendor/jquery-1.9.1.min.js
 ↳ jquery 1.9.1.min has known vulnerabilities: severity: medium; issue: 2432, summary: 3rd party CORS request may execute; https://github.com/jquery/jquery/issues/2432 http://blog.jquery.com/2016/01/08/jquery-2-2-and-1-12-released/
/javascript/vendor/jquery-migrate-1.1.1.min.js
 ↳ jquery-migrate 1.1.1.min has known vulnerabilities: severity: medium; release: jQuery Migrate 1.2.0 Released, summary: cross-site-scripting; http://blog.jquery.com/2013/05/01/jquery-migrate-1-2-0-released/ severity: medium; bug: 11290, summary: Selector interpreted as HTML; http://bugs.jquery.com/ticket/11290 http://research.insecurelabs.org/jquery/test/
/javascript/vendor/moment.min.js
 ↳ moment.js 2.10.6 has known vulnerabilities: severity: low; summary: reDOS - regular expression denial of service; https://github.com/moment/moment/issues/2936
```



Find your installed-packages.txt from your yocto build. For information on that see: [http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#understanding-what-the-build-history-contains](http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#understanding-what-the-build-history-contains)

**As of Yocto 2.2 Morty, a built-in** `cve-check` [**BitBake class**](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/classes/cve-check.bbclass) **was added to help automate checking of recipes against public CVEs at build time. See the following Yocto page for additional details:** [**https://docs.yoctoproject.org/dev/dev-manual/vulnerabilities.html**](https://docs.yoctoproject.org/dev/dev-manual/vulnerabilities.html)

### Yocto Project CVE Checking and Vulnerability Management (2024-2025)

The Yocto Project provides comprehensive built-in vulnerability management capabilities that have been significantly enhanced in recent releases. The **cve-check** infrastructure compares software packages against the NIST National Vulnerability Database (NVD) to identify known security issues.

#### Enabling CVE Checking in Yocto Builds

**Basic CVE Check Configuration (local.conf)**:
```bitbake
# Enable CVE checking for all recipes
INHERIT += "cve-check"

# Show warnings for CVEs that don't affect enabled features
CVE_CHECK_SHOW_WARNINGS = "1"

# Specify report format (text or JSON)
CVE_CHECK_REPORT_FORMAT = "json"

# Directory for CVE reports (default: tmp/deploy/cve)
CVE_CHECK_DIR = "${DEPLOY_DIR}/cve"
```

**Running CVE Check**:
```bash
# Run CVE check for specific image
bitbake core-image-minimal -c cve_check

# Run CVE check for all recipes
bitbake world -c cve_check
```

#### Understanding CVE Reports

After a build with CVE check enabled, reports for each compiled source recipe will be found in `build/tmp/deploy/cve/`. Each report includes:

1. **Metadata**: Recipe name, version, target, timestamp
2. **CVE Issues**: CVE ID, severity, description, NVD link
3. **Status**: Patched, Unpatched, or Ignored
4. **CVSS Scores**: Version 2 and Version 3 scores when available

**Example CVE Report (JSON)**:
```json
{
  "package": "openssl",
  "version": "3.0.8",
  "target": "core-image-minimal",
  "cves": [
    {
      "id": "CVE-2023-12345",
      "cvssv3": "7.5",
      "severity": "HIGH",
      "status": "Patched",
      "description": "Buffer overflow in SSL/TLS handshake",
      "link": "https://nvd.nist.gov/vuln/detail/CVE-2023-12345"
    },
    {
      "id": "CVE-2023-67890",
      "cvssv3": "5.3",
      "severity": "MEDIUM",
      "status": "Unpatched",
      "description": "Denial of service via malformed certificate"
    }
  ]
}
```

#### Advanced CVE Filtering (Linux Kernel)

The Yocto CVE scanner can be configured to upload a **Linux kernel .config file** to reduce false positives by excluding CVEs for features not compiled into your kernel:

```bitbake
# In your kernel recipe or local.conf
CVE_CHECK_KERNEL_CONFIG = "${B}/.config"
```

This dramatically reduces kernel CVE reports by filtering out vulnerabilities in kernel features/drivers that are not enabled in your build.

#### Ignoring Known CVEs

For CVEs that don't apply to your use case (e.g., server-only vulnerabilities in embedded device):

```bitbake
# In recipe (.bb file) or local.conf
CVE_CHECK_IGNORE += "CVE-2023-12345 CVE-2023-67890"
```

Document the justification in your recipe or security documentation:
```bitbake
# CVE-2023-12345: Affects Apache server module not used in embedded build
# CVE-2023-67890: Requires physical USB access; device has USB disabled
CVE_CHECK_IGNORE += "CVE-2023-12345 CVE-2023-67890"
```

#### Yocto CVE Infrastructure Updates (2024)

**Recent Enhancements**:
- **Scarthgap 5.0 LTS (May 2024)**: Enhanced CVE database sync, improved reporting
- **Enhanced QA Checks (2024)**: Automatic checking for SECURITY files in layers
- **Continuous CVE Updates**: Regular security patches for cups, curl, openssl, qemu, python3, vim
- **YP 5.2 (January 2025)**: Further CVE infrastructure improvements

#### Third-Party Yocto Vulnerability Tools

**meta-timesys (Vigiles Integration)**:
```bitbake
# Add meta-timesys to bblayers.conf
BBLAYERS += "/path/to/meta-timesys"

# In local.conf
INHERIT += "vigiles"
VIGILES_DASHBOARD_CONFIG = "https://linuxlink.timesys.com/..."
```

Features:
- Build-time CVE analysis and notifications
- SBOM generation for Vigiles platform
- Kernel .config-aware filtering
- Automated vulnerability monitoring

**meta-cvescan (Open Source Alternative)**:
```bash
# Clone meta-cvescan layer
git clone https://gitlab.com/akuster/meta-cvescan.git

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-cvescan"

# In local.conf
INHERIT += "cvescan"
```

Generates JSON inventory of all packages for vulnerability scanning.

**VulnScout (SBOM-based Vulnerability Scanner)**:
```bash
# Install VulnScout
pip install vulnscout

# Scan Yocto-generated SBOM
vulnscout scan --sbom yocto-sbom.spdx.json --format json
```

Pulls vulnerabilities from NVD, EPSS, and Grype databases.

#### CI/CD Integration Example

```yaml
# .gitlab-ci.yml - Yocto CVE checking in pipeline
yocto-cve-check:
  stage: security
  script:
    - source oe-init-build-env
    - bitbake core-image-minimal -c cve_check

    # Parse JSON reports
    - python3 scripts/parse_cve_reports.py tmp/deploy/cve/*.json

    # Fail if critical/high unpatched CVEs found
    - |
      CRITICAL=$(jq '[.cves[] | select(.severity=="CRITICAL" and .status=="Unpatched")] | length' tmp/deploy/cve/*.json)
      if [ "$CRITICAL" -gt 0 ]; then
        echo "❌ $CRITICAL critical unpatched CVEs found!"
        exit 1
      fi
  artifacts:
    paths:
      - tmp/deploy/cve/*.json
    expire_in: 30 days
  only:
    - main
    - merge_requests
```

**Considerations (Disclaimer: The List below is non-exhaustive):**

### Modern SBOM (Software Bill of Materials) Tools (2025)

A Software Bill of Materials is now critical for supply chain security and compliance. Modern SBOM formats and tools include:

#### SBOM Standards and Formats

1. **SPDX (Software Package Data Exchange)**
   * ISO/IEC 5962:2021 standard
   * Supported formats: JSON, YAML, RDF, XML, tag-value
   * Tools: [SPDX tools](https://github.com/spdx/tools-python), [sbom-tool](https://github.com/microsoft/sbom-tool)
   * Use case: Comprehensive license compliance and vulnerability tracking

2. **CycloneDX**
   * OWASP standard specifically designed for security use cases
   * JSON and XML formats
   * Tools: [CycloneDX CLI](https://github.com/CycloneDX/cyclonedx-cli), language-specific generators
   * Use case: Vulnerability management, software composition analysis

3. **SWID Tags (ISO/IEC 19770-2)**
   * Software identification tags
   * XML format
   * Use case: Software asset management

#### SBOM Generation Tools

**Multi-Language/Universal**:
* [**Syft**](https://github.com/anchore/syft) - Generate SBOMs from container images, filesystems, archives
  * Supports: SPDX, CycloneDX, Syft JSON formats
  * `syft packages dir:. -o spdx-json=sbom.spdx.json`
* [**Trivy**](https://github.com/aquasecurity/trivy) - Comprehensive scanner with SBOM generation
  * `trivy image --format spdx-json --output sbom.spdx.json myimage:latest`
* [**Tern**](https://github.com/tern-tools/tern) - Container inspection tool for SBOM generation
* [**FOSSA**](https://fossa.com/) - Commercial SBOM and license compliance platform

**Build System Specific**:
* **Yocto/OpenEmbedded**: See comprehensive Yocto SBOM section below
* **Buildroot**: Use `make legal-info` + convert to SPDX/CycloneDX
* **CMake**: [sbom-tool](https://github.com/microsoft/sbom-tool) integration
* **Rust/Cargo**: [cargo-sbom](https://github.com/psastras/sbom-rs)
* **Go**: [syft](https://github.com/anchore/syft), [cyclonedx-gomod](https://github.com/CycloneDX/cyclonedx-gomod)

### Yocto Project Native SBOM Generation (2024-2025)

The Yocto Project has **native, comprehensive SBOM generation capabilities** supporting both SPDX and CycloneDX formats. Recent releases have significantly enhanced SBOM functionality to meet modern supply chain security requirements and EU Cyber Resilience Act compliance.

#### SPDX SBOM Generation (ISO/IEC 5962:2021)

**SPDX 2.2 Format (Yocto 4.0+ Kirkstone)**:
```bitbake
# In local.conf
INHERIT += "create-spdx"

# Optional: Customize SPDX output directory
SPDX_DEPLOY_DIR = "${DEPLOY_DIR}/spdx"

# Optional: Include source code in SPDX documents
SPDX_INCLUDE_SOURCES = "1"
```

**SPDX 3.0.1 Format (Yocto 5.0+ Styhead - Default in 2024)**:
```bitbake
# In local.conf - SPDX 3.0.1 is now default in Styhead
INHERIT += "create-spdx-3.0"

# Generate both SPDX 2.2 and 3.0.1 for compatibility
INHERIT += "create-spdx create-spdx-3.0"
```

**Key Features**:
- **Per-recipe SPDX documents**: Each recipe generates its own SPDX file
- **Image-level SBOM**: Complete SBOM for final firmware image
- **License compliance**: SPDX includes comprehensive license information
- **Relationship tracking**: Documents dependencies between packages
- **Security metadata**: CVE information can be included

**SPDX Output Structure**:
```
tmp/deploy/spdx/
├── packages/
│   ├── openssl-3.0.8.spdx.json
│   ├── busybox-1.36.0.spdx.json
│   └── kernel-6.1.spdx.json
├── images/
│   └── core-image-minimal.spdx.json  # Complete image SBOM
└── packagedata/
    └── [package metadata]
```

#### CycloneDX SBOM Generation (OWASP Format)

While Yocto has native CycloneDX support planned, current approach uses conversion:

```bash
# Generate SPDX first
bitbake core-image-minimal

# Convert SPDX to CycloneDX using sbom-tool
sbom-tool convert --input tmp/deploy/spdx/images/core-image-minimal.spdx.json \
                  --output core-image-minimal.cyclonedx.json \
                  --format cyclonedx-1.5
```

**Alternative: meta-sbom layer** (if available):
```bitbake
# Add meta-sbom to bblayers.conf
BBLAYERS += "/path/to/meta-sbom"

# In local.conf
INHERIT += "cyclonedx"
```

#### buildhistory for Package Tracking

Yocto's buildhistory feature provides detailed package tracking for SBOM creation:

```bitbake
# In local.conf
INHERIT += "buildhistory"
BUILDHISTORY_COMMIT = "1"  # Commit changes to git repo
BUILDHISTORY_DIR = "${TOPDIR}/buildhistory"
BUILDHISTORY_FEATURES = "image package sdk"
```

This creates a git repository tracking:
- `installed-packages.txt`: All packages in image with versions
- `installed-package-sizes.txt`: Package size information
- `files-in-image.txt`: Complete file listing
- Recipe/package change history over time

**Access buildhistory data**:
```bash
# View installed packages
cat buildhistory/images/core-image-minimal/installed-packages.txt

# Example output:
# openssl 3.0.8-r0
# busybox 1.36.0-r0
# kernel-6.1.45-r0
# glibc 2.38-r0
```

#### Third-Party Yocto SBOM Tools

**meta-wr-sbom (Wind River)**:
```bash
# Clone meta-wr-sbom layer
git clone https://github.com/Wind-River/meta-wr-sbom.git

# Add to bblayers.conf
BBLAYERS += "/path/to/meta-wr-sbom"

# In local.conf
INHERIT += "wr-sbom"
WR_SBOM_FORMAT = "spdx"  # or "cyclonedx"
```

Features:
- SPDX v2.2 specification support
- Accurate component identification
- Security and licensing information
- Relationship mapping between components

**meta-timesys SBOM Integration**:
```bitbake
# In local.conf with meta-timesys
INHERIT += "vigiles"

# Generates SBOM for Timesys Vigiles platform
# Automatically includes CVE analysis
```

#### SBOM Validation and Quality Checks

**Validate SPDX SBOM**:
```bash
# Install SPDX tools
pip install spdx-tools

# Validate SPDX document
spdx-tools validate tmp/deploy/spdx/images/core-image-minimal.spdx.json

# Convert SPDX formats
spdx-tools convert --input core-image-minimal.spdx.json \
                   --output core-image-minimal.spdx.xml \
                   --format xml
```

**SBOM Quality Checks**:
- Verify all packages have CPE (Common Platform Enumeration) identifiers
- Check for missing license information
- Validate SBOM completeness (all files tracked)
- Ensure security metadata is included

#### EU Cyber Resilience Act (CRA) Compliance

Yocto SBOMs support CRA requirements:

1. **Product SBOM** (Annex I): SPDX documents provide complete component inventory
2. **Vulnerability Disclosure**: CVE data integrated into SBOM via cve-check class
3. **Update Mechanisms**: SBOM tracks component versions for security updates
4. **Support Period**: buildhistory tracks SBOM changes over product lifecycle

**CRA-Compliant SBOM Configuration**:
```bitbake
# In local.conf - Complete CRA compliance
INHERIT += "create-spdx-3.0 cve-check buildhistory"

# Include CVE data in SBOM
CVE_CHECK_SHOW_WARNINGS = "1"
CVE_CHECK_REPORT_FORMAT = "json"

# Track all changes
BUILDHISTORY_COMMIT = "1"
BUILDHISTORY_PUSH_REPO = "ssh://git@gitlab.com/company/sbom-history.git"

# Include source code provenance
SPDX_INCLUDE_SOURCES = "1"
```

#### CI/CD SBOM Automation

```yaml
# .gitlab-ci.yml - Automated SBOM generation and analysis
sbom-generation:
  stage: build
  script:
    - source oe-init-build-env
    - bitbake core-image-minimal

    # SBOMs generated in tmp/deploy/spdx
    - ls -lah tmp/deploy/spdx/images/

  artifacts:
    paths:
      - tmp/deploy/spdx/**/*.spdx.json
    expire_in: 1 year

sbom-vulnerability-scan:
  stage: security
  needs:
    - sbom-generation
  script:
    # Scan SBOM for vulnerabilities
    - grype sbom:tmp/deploy/spdx/images/core-image-minimal.spdx.json

    # Upload to Dependency-Track for continuous monitoring
    - |
      curl -X POST "https://dependency-track.company.com/api/v1/bom" \
           -H "X-Api-Key: $DTRACK_API_KEY" \
           -H "Content-Type: multipart/form-data" \
           -F "project=embedded-product" \
           -F "bom=@tmp/deploy/spdx/images/core-image-minimal.spdx.json"
  only:
    - main
    - tags
```

#### SBOM Best Practices for Yocto

1. **Generate SBOMs for every release build**: Track component changes over time
2. **Include both recipe-level and image-level SBOMs**: Complete traceability
3. **Use SPDX 3.0.1 for new projects**: Latest standard with enhanced security metadata
4. **Integrate CVE checking with SBOM generation**: Single source of truth
5. **Store SBOMs in version control**: Track SBOM evolution alongside code
6. **Automate SBOM validation**: Ensure quality and completeness
7. **Share SBOMs with customers**: Transparency for supply chain security

#### Vulnerability Scanning with SBOM

**Modern Vulnerability Scanners**:
* [**Grype**](https://github.com/anchore/grype) - Vulnerability scanner for SBOMs and container images
  * `grype sbom:./sbom.spdx.json`
* [**OSV-Scanner**](https://github.com/google/osv-scanner) - Google's vulnerability scanner
  * Uses OSV (Open Source Vulnerabilities) database
  * `osv-scanner --sbom=sbom.spdx.json`
* [**Trivy**](https://github.com/aquasecurity/trivy) - Multi-purpose security scanner
  * Scans: containers, filesystems, git repos, SBOMs
  * `trivy sbom sbom.spdx.json`
* [**Dependency-Track**](https://dependencytrack.org/) - Continuous SBOM analysis platform
  * Upload SBOMs, continuous vulnerability monitoring
  * Supports SPDX, CycloneDX, SWID

**Language-Specific Tools**:
* **JavaScript/Node.js**:
  * `npm audit` - Built-in npm vulnerability checker
  * [Retire.js](https://github.com/RetireJS/retire.js) - Detect vulnerable JS libraries
  * [Snyk](https://snyk.io/) - Commercial vulnerability scanner
* **Python**:
  * [pip-audit](https://github.com/pypa/pip-audit) - Official Python security auditor
  * [Safety](https://github.com/pyupio/safety) - Python dependency checker
  * `pip-audit --format cyclonedx-json --output sbom.json`
* **Rust**:
  * [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit)
  * `cargo audit --json`
* **C/C++**:
  * [OWASP Dependency-Check](https://github.com/jeremylong/DependencyCheck)
  * [Clair](https://github.com/quay/clair) - Container vulnerability scanner

#### Supply Chain Security Frameworks

**SLSA (Supply-chain Levels for Software Artifacts)**:
* Framework for ensuring software supply chain integrity
* Levels 1-4 with increasing security guarantees
* [SLSA Specification](https://slsa.dev/)
* Tools: [slsa-framework/slsa-verifier](https://github.com/slsa-framework/slsa-verifier)

**Sigstore**:
* Keyless signing and verification for software supply chain
* Components: Cosign, Rekor, Fulcio
* [Cosign](https://github.com/sigstore/cosign) - Sign and verify container images and SBOMs
  * `cosign sign-blob --bundle sbom.bundle sbom.spdx.json`
  * `cosign verify-blob --bundle sbom.bundle sbom.spdx.json`

**In-Toto**:
* Supply chain integrity framework
* Cryptographically verify who did what in the software supply chain
* [in-toto](https://in-toto.io/)

#### CI/CD Integration

**Example GitHub Actions workflow for SBOM generation**:
```yaml
name: Generate SBOM
on: [push]
jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          format: spdx-json
          output-file: sbom.spdx.json
      - name: Scan SBOM for vulnerabilities
        uses: anchore/scan-action@v3
        with:
          sbom: sbom.spdx.json
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.spdx.json
```

#### Embedded Linux Specific

**Yocto SBOM Generation**:
```bash
# In conf/local.conf
INHERIT += "create-spdx"
SPDX_PRETTY = "1"

# Build with SBOM generation
bitbake core-image-minimal

# SBOMs located in:
# tmp/deploy/spdx/<machine>/
```

**Buildroot with SBOM**:
```bash
# Generate legal info
make legal-info

# Convert to SPDX using tools
# Output in: output/legal-info/

# Use cyclonedx-cli or custom scripts to convert
```

#### Best Practices

1. **Automate SBOM Generation**:
   * Generate SBOMs in CI/CD pipeline
   * Version and archive SBOMs with releases
   * Update SBOMs with each build

2. **Continuous Vulnerability Monitoring**:
   * Scan SBOMs regularly (daily/weekly)
   * Integrate with vulnerability databases (NVD, OSV, GitHub Advisory)
   * Set up alerts for new vulnerabilities

3. **SBOM Distribution**:
   * Provide SBOMs to customers (transparency)
   * Sign SBOMs cryptographically (Sigstore)
   * Store SBOMs in artifact registry

4. **Compliance and Reporting**:
   * Map to compliance requirements (FDA, automotive, etc.)
   * Track open-source license obligations
   * Generate vulnerability reports for stakeholders

5. **Legacy Tools** (still useful):
   * [OWASP Dependency-Check](https://github.com/jeremylong/DependencyCheck)
   * [OWASP ZAP](https://www.zaproxy.org/) - Web application security testing
   * [Lynis](https://cisofy.com/lynis/) - System hardening auditing
   * [LibScanner](https://github.com/scriptingxss/LibScanner) - Yocto-specific CVE scanner

#### Compliance Requirements (2025)

* **US Executive Order 14028**: Federal software must have SBOM
* **EU Cyber Resilience Act (CRA)**: Manufacturers must maintain SBOM
* **NTIA Minimum Elements**: Essential SBOM components defined
* **FDA Medical Device Cybersecurity**: SBOM required for submissions

## Additional References <a href="#additional-references" id="additional-references"></a>

* [https://www.kb.cert.org/vuls/id/922681](https://www.kb.cert.org/vuls/id/922681)
* [https://www.kb.cert.org/vuls/id/561444](https://www.kb.cert.org/vuls/id/561444)
* [https://buildroot.org/downloads/manual/manual.html#faq-no-binary-packages](https://buildroot.org/downloads/manual/manual.html#faq-no-binary-packages)
* [https://wiki.yoctoproject.org/wiki/Security](https://wiki.yoctoproject.org/wiki/Security)
* [https://nvd.nist.gov/](https://nvd.nist.gov/)
* [https://www.openhub.net/](https://www.openhub.net/)
* [Improving Your Embedded Linux Security Posture with Yocto](https://legacy.gitbook.com/book/scriptingxss/embedded-appsec-best-practices/edit)
