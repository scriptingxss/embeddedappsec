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

### Yocto Project CVE Checking and Vulnerability Management

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

#### Yocto CVE Infrastructure Updates

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

### Modern SBOM Tools

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

### Yocto Project Native SBOM Generation

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

#### Compliance Requirements

* **US Executive Order 14028**: Federal software must have SBOM
* **EU Cyber Resilience Act (CRA)**: Manufacturers must maintain SBOM
* **NTIA Minimum Elements**: Essential SBOM components defined
* **FDA Medical Device Cybersecurity**: SBOM required for submissions

## Advanced SBOM Automation Workflows

Beyond basic SBOM generation, modern embedded development requires automated SBOM workflows that integrate with CI/CD pipelines, vulnerability databases, and compliance reporting systems.

### Multi-Format SBOM Generation Pipeline

**Automated SBOM generation for multiple formats**:

```yaml
# .gitlab-ci.yml - Comprehensive SBOM generation
stages:
  - build
  - sbom-generation
  - sbom-analysis
  - compliance-check

yocto-build:
  stage: build
  script:
    - source oe-init-build-env
    - echo 'INHERIT += "create-spdx-3.0 buildhistory"' >> conf/local.conf
    - bitbake core-image-minimal
  artifacts:
    paths:
      - tmp/deploy/spdx/**/*.spdx.json
      - buildhistory/
    expire_in: 30 days

sbom-multi-format:
  stage: sbom-generation
  needs:
    - yocto-build
  script:
    # Generate SPDX (already created by Yocto)
    - cp tmp/deploy/spdx/images/core-image-minimal.spdx.json sbom-spdx.json

    # Convert SPDX to CycloneDX
    - |
      docker run --rm -v $(pwd):/work cyclonedx/cyclonedx-cli \
        convert --input-file /work/sbom-spdx.json \
                --output-file /work/sbom-cyclonedx.json \
                --input-format spdxjson \
                --output-format json

    # Generate human-readable HTML report
    - |
      docker run --rm -v $(pwd):/work \
        ghcr.io/ckotzbauer/sbom-operator:latest \
        render --input /work/sbom-cyclonedx.json \
               --output /work/sbom-report.html

    # Generate CSV for spreadsheet analysis
    - |
      python3 << 'EOF'
      import json
      import csv

      with open('sbom-cyclonedx.json', 'r') as f:
          sbom = json.load(f)

      with open('sbom-inventory.csv', 'w', newline='') as csvfile:
          writer = csv.writer(csvfile)
          writer.writerow(['Name', 'Version', 'License', 'Supplier', 'CPE'])

          for component in sbom.get('components', []):
              writer.writerow([
                  component.get('name', 'N/A'),
                  component.get('version', 'N/A'),
                  ','.join([lic.get('id', 'N/A') for lic in component.get('licenses', [])]),
                  component.get('supplier', {}).get('name', 'N/A'),
                  component.get('cpe', 'N/A')
              ])
      EOF

  artifacts:
    paths:
      - sbom-spdx.json
      - sbom-cyclonedx.json
      - sbom-report.html
      - sbom-inventory.csv
    expire_in: 1 year

sbom-vulnerability-scan:
  stage: sbom-analysis
  needs:
    - sbom-multi-format
  script:
    # Scan with multiple tools for comprehensive coverage

    # 1. Grype scan
    - grype sbom:sbom-spdx.json -o json > grype-results.json
    - grype sbom:sbom-spdx.json -o table

    # 2. Trivy scan
    - trivy sbom sbom-spdx.json --format json --output trivy-results.json
    - trivy sbom sbom-spdx.json --format table

    # 3. OSV-Scanner
    - osv-scanner --sbom=sbom-spdx.json --format json --output osv-results.json

    # 4. Aggregate results
    - |
      python3 << 'EOF'
      import json
      from collections import defaultdict

      # Aggregate vulnerabilities from all scanners
      all_vulns = defaultdict(lambda: {'scanners': [], 'severity': 'UNKNOWN'})

      # Parse Grype results
      with open('grype-results.json', 'r') as f:
          grype = json.load(f)
          for match in grype.get('matches', []):
              vuln_id = match['vulnerability']['id']
              all_vulns[vuln_id]['scanners'].append('grype')
              all_vulns[vuln_id]['severity'] = match['vulnerability'].get('severity', 'UNKNOWN')

      # Parse Trivy results
      with open('trivy-results.json', 'r') as f:
          trivy = json.load(f)
          for result in trivy.get('Results', []):
              for vuln in result.get('Vulnerabilities', []):
                  vuln_id = vuln['VulnerabilityID']
                  all_vulns[vuln_id]['scanners'].append('trivy')
                  all_vulns[vuln_id]['severity'] = vuln.get('Severity', 'UNKNOWN')

      # Generate summary report
      critical = sum(1 for v in all_vulns.values() if v['severity'] == 'CRITICAL')
      high = sum(1 for v in all_vulns.values() if v['severity'] == 'HIGH')
      medium = sum(1 for v in all_vulns.values() if v['severity'] == 'MEDIUM')
      low = sum(1 for v in all_vulns.values() if v['severity'] == 'LOW')

      print(f"Vulnerability Summary:")
      print(f"  CRITICAL: {critical}")
      print(f"  HIGH: {high}")
      print(f"  MEDIUM: {medium}")
      print(f"  LOW: {low}")
      print(f"  Total: {len(all_vulns)}")

      # Fail if critical/high vulnerabilities found
      if critical > 0 or high > 0:
          print(f"\n❌ FAIL: Found {critical} critical and {high} high severity vulnerabilities")
          exit(1)
      EOF

  artifacts:
    when: always
    paths:
      - grype-results.json
      - trivy-results.json
      - osv-results.json
    reports:
      junit: vulnerability-report.xml

compliance-check:
  stage: compliance-check
  needs:
    - sbom-multi-format
  script:
    # Validate SBOM completeness
    - |
      python3 << 'EOF'
      import json
      import sys

      with open('sbom-cyclonedx.json', 'r') as f:
          sbom = json.load(f)

      # NTIA Minimum Elements check
      errors = []

      # 1. Author name
      if not sbom.get('metadata', {}).get('authors'):
          errors.append("Missing author information")

      # 2. Timestamp
      if not sbom.get('metadata', {}).get('timestamp'):
          errors.append("Missing timestamp")

      # 3. Component name, version, supplier
      for component in sbom.get('components', []):
          if not component.get('name'):
              errors.append(f"Component missing name")
          if not component.get('version'):
              errors.append(f"Component {component.get('name')} missing version")
          if not component.get('supplier') and not component.get('publisher'):
              errors.append(f"Component {component.get('name')} missing supplier")

      # 4. Dependency relationships
      if not sbom.get('dependencies'):
          errors.append("Missing dependency relationships")

      if errors:
          print("SBOM Compliance Errors:")
          for error in errors:
              print(f"  - {error}")
          sys.exit(1)
      else:
          print("✅ SBOM meets NTIA minimum elements")
      EOF

    # License compliance check
    - |
      python3 << 'EOF'
      import json

      with open('sbom-cyclonedx.json', 'r') as f:
          sbom = json.load(f)

      # Forbidden licenses (copyleft for proprietary products)
      forbidden = ['GPL-2.0', 'GPL-3.0', 'AGPL-3.0']
      violations = []

      for component in sbom.get('components', []):
          licenses = component.get('licenses', [])
          for lic in licenses:
              lic_id = lic.get('license', {}).get('id', '')
              if any(forbidden_lic in lic_id for forbidden_lic in forbidden):
                  violations.append(f"{component['name']}: {lic_id}")

      if violations:
          print("⚠️  License compliance violations:")
          for v in violations:
              print(f"  - {v}")
      else:
          print("✅ No license violations found")
      EOF

  allow_failure: false
```

### Dependency-Track Integration

**Continuous SBOM monitoring with Dependency-Track**:

```yaml
# .gitlab-ci.yml - Dependency-Track upload
dependency-track-upload:
  stage: sbom-analysis
  needs:
    - sbom-multi-format
  script:
    # Upload SBOM to Dependency-Track for continuous monitoring
    - |
      PROJECT_UUID=$(curl -s -X GET "${DTRACK_URL}/api/v1/project/lookup?name=embedded-device&version=1.0" \
        -H "X-Api-Key: ${DTRACK_API_KEY}" | jq -r '.uuid')

      if [ "$PROJECT_UUID" == "null" ]; then
        # Create new project
        PROJECT_UUID=$(curl -s -X PUT "${DTRACK_URL}/api/v1/project" \
          -H "Content-Type: application/json" \
          -H "X-Api-Key: ${DTRACK_API_KEY}" \
          -d '{
            "name": "embedded-device",
            "version": "1.0",
            "classifier": "APPLICATION"
          }' | jq -r '.uuid')
      fi

      # Upload SBOM
      curl -X POST "${DTRACK_URL}/api/v1/bom" \
        -H "Content-Type: multipart/form-data" \
        -H "X-Api-Key: ${DTRACK_API_KEY}" \
        -F "project=${PROJECT_UUID}" \
        -F "bom=@sbom-cyclonedx.json"

    # Wait for analysis to complete
    - sleep 30

    # Fetch vulnerability results
    - |
      FINDINGS=$(curl -s -X GET "${DTRACK_URL}/api/v1/finding/project/${PROJECT_UUID}" \
        -H "X-Api-Key: ${DTRACK_API_KEY}")

      CRITICAL=$(echo "$FINDINGS" | jq '[.[] | select(.vulnerability.severity=="CRITICAL")] | length')
      HIGH=$(echo "$FINDINGS" | jq '[.[] | select(.vulnerability.severity=="HIGH")] | length')

      echo "Dependency-Track Results:"
      echo "  Critical: $CRITICAL"
      echo "  High: $HIGH"

      if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
        echo "❌ FAIL: Critical or high vulnerabilities found in Dependency-Track"
        exit 1
      fi
```

**Dependency-Track Yocto Recipe**:

```bitbake
# dependency-track_4.9.bb
DESCRIPTION = "Dependency-Track - SBOM vulnerability analysis platform"
LICENSE = "Apache-2.0"

SRC_URI = "https://github.com/DependencyTrack/dependency-track/releases/download/${PV}/dependency-track-apiserver.jar \
           file://dependency-track.service"

inherit systemd java

RDEPENDS:${PN} = "openjdk-11"

do_install() {
    install -d ${D}${datadir}/dependency-track
    install -m 0644 ${WORKDIR}/dependency-track-apiserver.jar ${D}${datadir}/dependency-track/

    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/dependency-track.service ${D}${systemd_system_unitdir}/

    install -d ${D}${sysconfdir}/dependency-track
    cat > ${D}${sysconfdir}/dependency-track/application.properties << 'EOF'
alpine.database.mode=external
alpine.database.url=jdbc:postgresql://localhost:5432/dtrack
alpine.database.driver=org.postgresql.Driver
alpine.database.username=dtrack
alpine.database.password=CHANGE_ME

# Enable API authentication
alpine.api.key.required=true
EOF
}

SYSTEMD_SERVICE:${PN} = "dependency-track.service"
```

### GitHub/GitLab Security Scanning Integration

**GitHub Actions Workflow**:

```yaml
# .github/workflows/sbom-security.yml
name: SBOM Generation and Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  sbom-generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          format: spdx-json
          output-file: sbom-spdx.json

      - name: Generate CycloneDX SBOM
        run: |
          docker run --rm -v $(pwd):/work cyclonedx/cyclonedx-cli \
            convert --input-file /work/sbom-spdx.json \
                    --output-file /work/sbom-cyclonedx.json \
                    --input-format spdxjson \
                    --output-format json

      - name: Upload SBOM artifacts
        uses: actions/upload-artifact@v3
        with:
          name: sbom-files
          path: |
            sbom-spdx.json
            sbom-cyclonedx.json

  vulnerability-scan:
    runs-on: ubuntu-latest
    needs: sbom-generate
    steps:
      - name: Download SBOM
        uses: actions/download-artifact@v3
        with:
          name: sbom-files

      - name: Scan with Grype
        uses: anchore/scan-action@v3
        with:
          sbom: sbom-spdx.json
          fail-build: true
          severity-cutoff: high

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'sbom'
          input: sbom-spdx.json
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  dependency-track-upload:
    runs-on: ubuntu-latest
    needs: sbom-generate
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download SBOM
        uses: actions/download-artifact@v3
        with:
          name: sbom-files

      - name: Upload to Dependency-Track
        env:
          DTRACK_URL: ${{ secrets.DTRACK_URL }}
          DTRACK_API_KEY: ${{ secrets.DTRACK_API_KEY }}
        run: |
          curl -X POST "${DTRACK_URL}/api/v1/bom" \
            -H "Content-Type: multipart/form-data" \
            -H "X-Api-Key: ${DTRACK_API_KEY}" \
            -F "project=embedded-device" \
            -F "projectVersion=1.0" \
            -F "bom=@sbom-cyclonedx.json"
```

### SBOM Differential Analysis

**Detect supply chain changes between releases**:

```python
#!/usr/bin/env python3
# sbom-diff.py - Compare SBOMs between releases

import json
import sys
from typing import Set, Dict, Tuple

def load_sbom(filename: str) -> Dict:
    with open(filename, 'r') as f:
        return json.load(f)

def extract_components(sbom: Dict) -> Set[Tuple[str, str]]:
    """Extract (name, version) tuples from SBOM"""
    components = set()
    for component in sbom.get('components', []):
        name = component.get('name', 'unknown')
        version = component.get('version', 'unknown')
        components.add((name, version))
    return components

def compare_sboms(old_sbom: Dict, new_sbom: Dict):
    old_components = extract_components(old_sbom)
    new_components = extract_components(new_sbom)

    added = new_components - old_components
    removed = old_components - new_components
    common = old_components & new_components

    print("SBOM Differential Analysis")
    print("=" * 50)
    print(f"Total components (old): {len(old_components)}")
    print(f"Total components (new): {len(new_components)}")
    print(f"Added: {len(added)}")
    print(f"Removed: {len(removed)}")
    print(f"Unchanged: {len(common)}")
    print()

    if added:
        print("Added Components:")
        for name, version in sorted(added):
            print(f"  + {name}@{version}")
        print()

    if removed:
        print("Removed Components:")
        for name, version in sorted(removed):
            print(f"  - {name}@{version}")
        print()

    # Check for version changes
    old_by_name = {name: version for name, version in old_components}
    new_by_name = {name: version for name, version in new_components}

    version_changes = []
    for name in old_by_name:
        if name in new_by_name and old_by_name[name] != new_by_name[name]:
            version_changes.append((name, old_by_name[name], new_by_name[name]))

    if version_changes:
        print("Version Changes:")
        for name, old_ver, new_ver in sorted(version_changes):
            print(f"  ~ {name}: {old_ver} → {new_ver}")
        print()

    # Risk assessment
    risk_score = 0
    if len(added) > 10:
        risk_score += 2
        print("⚠️  HIGH RISK: Many new components added (supply chain expansion)")
    if len(removed) > 10:
        risk_score += 1
        print("⚠️  MEDIUM RISK: Many components removed")
    if len(version_changes) > 20:
        risk_score += 1
        print("⚠️  MEDIUM RISK: Many version updates")

    if risk_score == 0:
        print("✅ LOW RISK: Minimal supply chain changes")

    return risk_score

if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Usage: sbom-diff.py <old-sbom.json> <new-sbom.json>")
        sys.exit(1)

    old_sbom = load_sbom(sys.argv[1])
    new_sbom = load_sbom(sys.argv[2])

    risk = compare_sboms(old_sbom, new_sbom)
    sys.exit(min(risk, 2))  # Exit code 0=low risk, 1=medium, 2=high
```

**Integration in CI/CD**:

```yaml
sbom-diff-analysis:
  stage: compliance-check
  script:
    # Download previous release SBOM
    - curl -o sbom-previous.json "${ARTIFACT_SERVER}/releases/v1.0/sbom.json"

    # Compare with current build
    - python3 sbom-diff.py sbom-previous.json sbom-cyclonedx.json

    # Fail if high risk
    - |
      RISK_CODE=$?
      if [ $RISK_CODE -eq 2 ]; then
        echo "❌ HIGH RISK supply chain changes detected"
        exit 1
      fi
```

## Continuous Vulnerability Monitoring

### Automated CVE Alert System

**Real-time CVE monitoring for deployed devices**:

```python
#!/usr/bin/env python3
# cve-monitor.py - Monitor deployed SBOMs for new CVEs

import json
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime, timedelta

class CVEMonitor:
    def __init__(self, sbom_path: str, nvd_api_key: str):
        self.sbom_path = sbom_path
        self.nvd_api_key = nvd_api_key
        self.nvd_url = "https://services.nvd.nist.gov/rest/json/cves/2.0"

    def load_sbom(self):
        with open(self.sbom_path, 'r') as f:
            sbom = json.load(f)

        components = []
        for component in sbom.get('components', []):
            cpe = component.get('cpe')
            if cpe:
                components.append({
                    'name': component.get('name'),
                    'version': component.get('version'),
                    'cpe': cpe
                })
        return components

    def check_recent_cves(self, cpe: str, days: int = 7):
        """Check NVD for CVEs published in last N days"""
        pub_start = (datetime.now() - timedelta(days=days)).isoformat()
        pub_end = datetime.now().isoformat()

        params = {
            'cpeName': cpe,
            'pubStartDate': pub_start,
            'pubEndDate': pub_end
        }
        headers = {
            'apiKey': self.nvd_api_key
        }

        response = requests.get(self.nvd_url, params=params, headers=headers)
        if response.status_code == 200:
            data = response.json()
            return data.get('vulnerabilities', [])
        return []

    def monitor(self):
        components = self.load_sbom()
        new_vulnerabilities = []

        print(f"Monitoring {len(components)} components for new CVEs...")

        for component in components:
            cpe = component['cpe']
            cves = self.check_recent_cves(cpe, days=7)

            if cves:
                for cve_item in cves:
                    cve = cve_item['cve']
                    new_vulnerabilities.append({
                        'component': component['name'],
                        'version': component['version'],
                        'cve_id': cve['id'],
                        'description': cve.get('descriptions', [{}])[0].get('value', 'N/A'),
                        'severity': cve.get('metrics', {}).get('cvssMetricV31', [{}])[0].get('cvssData', {}).get('baseSeverity', 'UNKNOWN')
                    })

        return new_vulnerabilities

    def send_alert(self, vulnerabilities: list, recipients: list):
        if not vulnerabilities:
            print("No new vulnerabilities found")
            return

        msg = MIMEMultipart()
        msg['From'] = 'cve-monitor@company.com'
        msg['To'] = ', '.join(recipients)
        msg['Subject'] = f'⚠️ New CVEs Detected in Deployed Firmware ({len(vulnerabilities)} found)'

        body = "New vulnerabilities detected in deployed firmware components:\n\n"

        for vuln in vulnerabilities:
            body += f"Component: {vuln['component']} {vuln['version']}\n"
            body += f"CVE: {vuln['cve_id']}\n"
            body += f"Severity: {vuln['severity']}\n"
            body += f"Description: {vuln['description'][:200]}...\n"
            body += f"Link: https://nvd.nist.gov/vuln/detail/{vuln['cve_id']}\n"
            body += "-" * 80 + "\n\n"

        msg.attach(MIMEText(body, 'plain'))

        # Send email
        with smtplib.SMTP('smtp.company.com', 587) as server:
            server.starttls()
            server.login('cve-monitor@company.com', 'password')
            server.send_message(msg)

        print(f"Alert sent to {recipients}")

if __name__ == '__main__':
    monitor = CVEMonitor(
        sbom_path='/path/to/deployed-sbom.json',
        nvd_api_key='YOUR_NVD_API_KEY'
    )

    vulnerabilities = monitor.monitor()

    if vulnerabilities:
        monitor.send_alert(
            vulnerabilities,
            recipients=['security@company.com', 'ops@company.com']
        )
```

**Cron-based CVE Monitoring**:

```bash
# /etc/cron.daily/cve-monitor.sh
#!/bin/bash

# Monitor deployed firmware for new CVEs
python3 /opt/cve-monitor/cve-monitor.py

# Check exit status
if [ $? -ne 0 ]; then
    echo "CVE monitoring failed" | mail -s "CVE Monitor Error" ops@company.com
fi
```

### VEX (Vulnerability Exploitability eXchange) Support

**Generate VEX documents to communicate vulnerability status**:

```python
#!/usr/bin/env python3
# generate-vex.py - Create VEX document for known CVEs

import json
from datetime import datetime

def generate_vex(sbom_path: str, cve_analysis: dict):
    """
    Generate CycloneDX VEX document

    cve_analysis: {
        'CVE-2023-12345': {
            'status': 'not_affected',  # or 'affected', 'fixed', 'under_investigation'
            'justification': 'component_not_present',
            'detail': 'Vulnerable function is not compiled in our build'
        }
    }
    """

    with open(sbom_path, 'r') as f:
        sbom = json.load(f)

    vex = {
        "bomFormat": "CycloneDX",
        "specVersion": "1.5",
        "version": 1,
        "metadata": {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "component": sbom['metadata']['component']
        },
        "vulnerabilities": []
    }

    for cve_id, analysis in cve_analysis.items():
        vuln_entry = {
            "id": cve_id,
            "source": {
                "name": "NVD",
                "url": f"https://nvd.nist.gov/vuln/detail/{cve_id}"
            },
            "analysis": {
                "state": analysis['status'],
                "justification": analysis.get('justification', 'code_not_reachable'),
                "detail": analysis['detail'],
                "response": ["update" if analysis['status'] == 'fixed' else "will_not_fix"]
            }
        }
        vex['vulnerabilities'].append(vuln_entry)

    return vex

if __name__ == '__main__':
    # Example: Document that CVE-2023-12345 doesn't affect our build
    cve_analysis = {
        'CVE-2023-12345': {
            'status': 'not_affected',
            'justification': 'vulnerable_code_not_in_execute_path',
            'detail': 'The vulnerable SSL_read() function is not used in our configuration. We use TLS 1.3 only.'
        },
        'CVE-2023-67890': {
            'status': 'fixed',
            'justification': 'patch_applied',
            'detail': 'Patched in openssl-3.0.8-r1 Yocto recipe'
        }
    }

    vex = generate_vex('sbom-cyclonedx.json', cve_analysis)

    with open('firmware-vex.json', 'w') as f:
        json.dump(vex, f, indent=2)

    print("VEX document generated: firmware-vex.json")
```

## OWASP IoT Ecosystem Integration

### Supply Chain Security Mapping

* **OWASP ISVS**: V6.1.1 - Software Bill of Materials is maintained and available
* **OWASP ISVS**: V6.1.2 - Third-party components are tracked and monitored for vulnerabilities
* **OWASP ISVS**: V6.2.1 - Cryptographic signing is used to verify software integrity
* **OWASP ISTG**: ISTG-FW-INFO-002 - Identify third-party components and versions
* **OWASP FSTM**: Stage 2 - Obtain firmware and extract SBOM information
* **OWASP IoTGoat**: Practice firmware extraction and component identification

### Compliance Framework Alignment

**EU Cyber Resilience Act (CRA)**:
- Annex I: Product SBOM required
- Article 11: Security updates for product lifetime
- Article 14: Vulnerability disclosure requirements

**US Executive Order 14028**:
- Section 4(e): SBOM required for federal software
- NTIA minimum elements compliance
- Continuous vulnerability monitoring

**FDA Medical Device Cybersecurity**:
- Premarket: SBOM submission required
- Postmarket: Vulnerability management and patching
- Transparency logs for security updates

## Additional References <a href="#additional-references" id="additional-references"></a>

* [https://www.kb.cert.org/vuls/id/922681](https://www.kb.cert.org/vuls/id/922681)
* [https://www.kb.cert.org/vuls/id/561444](https://www.kb.cert.org/vuls/id/561444)
* [https://buildroot.org/downloads/manual/manual.html#faq-no-binary-packages](https://buildroot.org/downloads/manual/manual.html#faq-no-binary-packages)
* [https://wiki.yoctoproject.org/wiki/Security](https://wiki.yoctoproject.org/wiki/Security)
* [https://nvd.nist.gov/](https://nvd.nist.gov/)
* [https://www.openhub.net/](https://www.openhub.net/)
* [Improving Your Embedded Linux Security Posture with Yocto](https://legacy.gitbook.com/book/scriptingxss/embedded-appsec-best-practices/edit)
