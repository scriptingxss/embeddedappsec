# Transport Layer Security

Ensure all methods of communication are utilizing industry standard encryption configurations for [TLS](https://wiki.sei.cmu.edu/confluence/display/c/API10-C.+APIs+should+have+security+options+enabled+by+default). The use of TLS ensures that all data remains confidential and untampered with while in transit. Utilize free certificate authority services such as [Let’s Encrypt](https://letsencrypt.org/) if the embedded device utilizes domain names.

[**Example**](http://fm4dd.com/openssl/certverify.htm) **of how to perform a basic certificate validation against a root certificate authority, using the OpenSSL library functions. :**

```c
#include <openssl/bio.h>
#include <openssl/err.h>
#include <openssl/pem.h>
#include <openssl/x509.h>
#include <openssl/x509_vfy.h>

int main() {

  const char ca_bundlestr[] = "./ca-bundle.pem";
  const char cert_filestr[] = "./cert-file.pem";

  BIO              *certbio = NULL;
  BIO               *outbio = NULL;
  X509          *error_cert = NULL;
  X509                *cert = NULL;
  X509_NAME    *certsubject = NULL;
  X509_STORE         *store = NULL;
  X509_STORE_CTX  *vrfy_ctx = NULL;
  int ret;

  /* ---------------------------------------------------------- *
   * These function calls initialize openssl for correct work.  *
   * ---------------------------------------------------------- */
  OpenSSL_add_all_algorithms();
  ERR_load_BIO_strings();
  ERR_load_crypto_strings();

  /* ---------------------------------------------------------- *
   * Create the Input/Output BIO's.                             *
   * ---------------------------------------------------------- */
  certbio = BIO_new(BIO_s_file());
  outbio  = BIO_new_fp(stdout, BIO_NOCLOSE);

  /* ---------------------------------------------------------- *
   * Initialize the global certificate validation store object. *
   * ---------------------------------------------------------- */
  if (!(store=X509_STORE_new()))
     BIO_printf(outbio, "Error creating X509_STORE_CTX object\n");

  /* ---------------------------------------------------------- *
   * Create the context structure for the validation operation. *
   * ---------------------------------------------------------- */
  vrfy_ctx = X509_STORE_CTX_new();

  /* ---------------------------------------------------------- *
   * Load the certificate and cacert chain from file (PEM).     *
   * ---------------------------------------------------------- */
  ret = BIO_read_filename(certbio, cert_filestr);
  if (! (cert = PEM_read_bio_X509(certbio, NULL, 0, NULL))) {
    BIO_printf(outbio, "Error loading cert into memory\n");
    exit(-1);
  }

  ret = X509_STORE_load_locations(store, ca_bundlestr, NULL);
  if (ret != 1)
    BIO_printf(outbio, "Error loading CA cert or chain file\n");

  /* ---------------------------------------------------------- *
   * Initialize the ctx structure for a verification operation: *
   * Set the trusted cert store, the unvalidated cert, and any  *
   * potential certs that could be needed (here we set it NULL) *
   * ---------------------------------------------------------- */
  X509_STORE_CTX_init(vrfy_ctx, store, cert, NULL);

  /* ---------------------------------------------------------- *
   * Check the complete cert chain can be build and validated.  *
   * Returns 1 on success, 0 on verification failures, and -1   *
   * for trouble with the ctx object (i.e. missing certificate) *
   * ---------------------------------------------------------- */
  ret = X509_verify_cert(vrfy_ctx);
  BIO_printf(outbio, "Verification return code: %d\n", ret);

  if(ret == 0 || ret == 1)
  BIO_printf(outbio, "Verification result text: %s\n",
             X509_verify_cert_error_string(vrfy_ctx->error));

  /* ---------------------------------------------------------- *
   * The error handling below shows how to get failure details  *
   * from the offending certificate.                            *
   * ---------------------------------------------------------- */
  if(ret == 0) {
    /*  get the offending certificate causing the failure */
    error_cert  = X509_STORE_CTX_get_current_cert(vrfy_ctx);
    certsubject = X509_NAME_new();
    certsubject = X509_get_subject_name(error_cert);
    BIO_printf(outbio, "Verification failed cert:\n");
    X509_NAME_print_ex(outbio, certsubject, 0, XN_FLAG_MULTILINE);
    BIO_printf(outbio, "\n");
  }

  /* ---------------------------------------------------------- *
   * Free up all structures                                     *
   * ---------------------------------------------------------- */
  X509_STORE_CTX_free(vrfy_ctx);
  X509_STORE_free(store);
  X509_free(cert);
  BIO_free_all(certbio);
  BIO_free_all(outbio);
  exit(0);
}
```

**Considerations \(Disclaimer: The List below is non-exhaustive\):**

* **Use TLS 1.3** (RFC 8446) for all new embedded products as of 2025
  * TLS 1.3 provides improved security and performance over TLS 1.2
  * Removes legacy cryptographic algorithms and reduces handshake round trips
  * Mandatory for compliance with modern security frameworks (PCI DSS 4.0, etc.)
* **Minimum TLS version policy**:
  * New products: TLS 1.3 only (preferred)
  * Legacy support: TLS 1.2 minimum, disable TLS 1.0/1.1 and all SSL versions
* **Cipher suite selection** (TLS 1.3):
  * `TLS_AES_256_GCM_SHA384` (preferred)
  * `TLS_AES_128_GCM_SHA256`
  * `TLS_CHACHA20_POLY1305_SHA256`
  * Avoid: All CBC mode ciphers, RC4, 3DES, NULL ciphers
* **Cipher suite selection** (TLS 1.2 - for legacy compatibility):
  * `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
  * `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
  * `TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256`
* Consider implementing **mutual TLS (mTLS)** authentication for firmware that accepts TLS connections from a limited group of allowed clients.
* **Certificate validation requirements**:
  * Validate the complete certificate chain against trusted root CAs
  * Verify hostname matches certificate Common Name (CN) or Subject Alternative Name (SAN)
  * Check certificate expiration dates
  * Implement Certificate Revocation List (CRL) or Online Certificate Status Protocol (OCSP) checking
  * Validate certificate purpose and key usage extensions
* **Certificate and key management**:
  * Use SHA-256 or SHA-384 for certificate signing (SHA-1 is deprecated)
  * Minimum RSA key size: 2048 bits (3072+ bits recommended for long-term use)
  * Consider ECDSA P-256 or P-384 for resource-constrained devices
  * **Store private keys securely**:
    * Hardware Security Module (HSM) - ideal for production environments
    * Trusted Execution Environment (TEE) - e.g., ARM TrustZone, Intel SGX
    * Secure Element (SE) - for IoT devices
    * Encrypted key storage as minimum fallback
* **Post-Quantum Cryptography (PQC) readiness**:
  * Monitor NIST PQC standardization (Kyber, Dilithium)
  * Plan for hybrid classical/PQC deployments
  * Ensure firmware update capability for cryptographic algorithm migration
* **Certificate lifecycle management**:
  * Implement automated certificate renewal before expiration
  * Support certificate rotation without device reboot
  * Monitor certificate expiration (alert 60+ days before expiry)
  * Use short-lived certificates (90 days or less) where practical
* **Perfect Forward Secrecy (PFS)**:
  * Use ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) key exchange
  * Avoid RSA key exchange (does not provide PFS)
* **TLS configuration testing**:
  * Use [testssl.sh](https://testssl.sh/) for comprehensive TLS/SSL testing
  * Use [ssllabs.com](https://www.ssllabs.com) for server configuration analysis
  * Use nmap with `--script ssl-enum-ciphers.nse` for cipher suite enumeration
  * Consider automated tools: sslyze, sslscan, OpenSSL s_client
* **Additional security headers** (for TLS-protected web interfaces):
  * Strict-Transport-Security (HSTS) with long max-age
  * HTTP/2 or HTTP/3 (QUIC) for improved performance
* **Compliance and standards**:
  * Follow Mozilla SSL Configuration Generator recommendations
  * Comply with NIST SP 800-52 Rev. 2 (TLS guidelines)
  * Meet PCI DSS requirements (TLS 1.2+ mandatory since June 2023, TLS 1.3 recommended)
  * Consider FIPS 140-2/140-3 compliance for government/regulated industries

## Post-Quantum Cryptography for TLS (2025+)

### Quantum Threat to TLS

**Current TLS Vulnerabilities**:
- RSA key exchange: Broken by Shor's algorithm
- ECDHE key exchange: Broken by Shor's algorithm
- Certificate signatures (RSA/ECDSA): Broken by Shor's algorithm

**Harvest Now, Decrypt Later (HNDL)**:
- Attackers record TLS 1.3 traffic today
- Decrypt with quantum computer in 10-15 years
- Critical for long-lived embedded devices (industrial IoT, medical, automotive)

### Hybrid TLS 1.3 with PQC

**Standards**:
- IETF RFC 9180: Hybrid Public Key Encryption (HPKE)
- IETF Draft: Hybrid key exchange in TLS 1.3
- Cloudflare/Google: Production PQC TLS testing

**Hybrid Key Exchange**:
```
TLS 1.3 Key Exchange = Classical ECDHE + PQC KEM

Example:
- X25519 (ECDHE) + ML-KEM-768 → Hybrid shared secret
- Secure if either algorithm is secure
```

### OpenSSL 3.x with liboqs (PQC)

**Installation**:
```bash
# Install liboqs
git clone https://github.com/open-quantum-safe/liboqs.git
cd liboqs && mkdir build && cd build
cmake -DOQS_USE_OPENSSL=ON ..
make && sudo make install

# Install OQS-OpenSSL provider
git clone https://github.com/open-quantum-safe/oqs-provider.git
cd oqs-provider && mkdir build && cd build
cmake -DOPENSSL_ROOT_DIR=/usr/local/ssl ..
make && sudo make install
```

**Configuration**:
```bash
# Enable OQS provider in openssl.cnf
[openssl_init]
providers = provider_sect

[provider_sect]
default = default_sect
oqsprovider = oqsprovider_sect

[oqsprovider_sect]
activate = 1
```

### Hybrid TLS Server Configuration

**Example: Embedded HTTPS Server with PQC**:
```c
#include <openssl/ssl.h>
#include <openssl/err.h>

SSL_CTX* create_pqc_context() {
    SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());

    // Load hybrid certificate (ECDSA + ML-DSA)
    SSL_CTX_use_certificate_file(ctx, "hybrid_cert.pem", SSL_FILETYPE_PEM);
    SSL_CTX_use_PrivateKey_file(ctx, "hybrid_key.pem", SSL_FILETYPE_PEM);

    // Set hybrid key exchange: X25519 + ML-KEM-768
    SSL_CTX_set1_groups_list(ctx, "x25519_mlkem768:x25519:prime256v1");

    // Set hybrid signature algorithms
    SSL_CTX_set1_sigalgs_list(ctx,
        "ecdsa_secp384r1_sha384:mldsa65:rsa_pss_rsae_sha256");

    return ctx;
}

int main() {
    SSL_CTX *ctx = create_pqc_context();

    // Create TLS connection with PQC protection
    int server_fd = create_socket(8443);
    SSL *ssl = SSL_new(ctx);
    SSL_set_fd(ssl, accept(server_fd, NULL, NULL));

    if (SSL_accept(ssl) > 0) {
        printf("PQC TLS handshake successful!\n");
        printf("Cipher: %s\n", SSL_get_cipher(ssl));

        // Communication protected by hybrid PQC+classical crypto
        char buf[1024];
        SSL_read(ssl, buf, sizeof(buf));
        SSL_write(ssl, "HTTP/1.1 200 OK\r\n\r\n", 19);
    }

    SSL_free(ssl);
    SSL_CTX_free(ctx);
    return 0;
}
```

### wolfSSL with PQC Support

**Experimental PQC Build**:
```bash
# Build wolfSSL with liboqs integration
./configure --enable-pqc --with-liboqs=/usr/local
make && sudo make install
```

**Client Example**:
```c
#include <wolfssl/ssl.h>

int connect_pqc_server(const char *host) {
    wolfSSL_Init();
    WOLFSSL_CTX *ctx = wolfSSL_CTX_new(wolfTLSv1_3_client_method());

    // Enable hybrid groups
    wolfSSL_CTX_set_groups(ctx, (int[]){
        WOLFSSL_ECC_X25519,
        WOLFSSL_PQC_MLKEM768
    }, 2);

    WOLFSSL *ssl = wolfSSL_new(ctx);
    wolfSSL_connect(ssl);

    return 0;
}
```

### Performance Considerations

**Handshake Overhead**:
- Classical ECDHE: ~1ms
- Hybrid X25519 + ML-KEM-768: ~3-5ms
- **Impact**: Acceptable for most embedded use cases

**Bandwidth Overhead**:
- ML-KEM-768 public key: 1184 bytes
- ML-KEM-768 ciphertext: 1088 bytes
- Total TLS handshake increase: ~2.3KB

**Mitigation**:
- Use TLS 1.3 session resumption (0-RTT)
- Consider ML-KEM-512 for IoT (smaller keys)
- Hardware acceleration where available

### Certificate Authority PQC Readiness

**Hybrid Certificates**:
- Wait for CA support (Let's Encrypt, DigiCert)
- Expected: 2025-2026 for production PQC CA
- Interim: Self-signed hybrid certificates for testing

**Certificate Chain**:
```
Root CA (RSA + ML-DSA)
  └─ Intermediate CA (ECDSA + ML-DSA)
      └─ Device Certificate (ECDSA + ML-DSA)
```

### Testing PQC TLS

**Tools**:
```bash
# Test with OQS-OpenSSL s_client
openssl s_client -connect device.local:8443 \
    -groups x25519_mlkem768

# Verify hybrid key exchange
openssl s_client -connect device.local:8443 \
    -showcerts -debug -msg
```

### Migration Roadmap

**2025-2026: Hybrid Testing**
- Deploy hybrid PQC+classical TLS in test environments
- Monitor performance and compatibility
- Build internal CA infrastructure with PQC support

**2026-2027: Production Hybrid**
- Roll out hybrid TLS to production devices
- Maintain classical-only fallback for legacy clients
- Update all embedded TLS libraries (OpenSSL, wolfSSL, mbedTLS)

**2028+: PQC-Only**
- Transition to PQC-only TLS
- Deprecate classical-only algorithms
- Full quantum-resistant communication

### OWASP IoT Ecosystem Integration

**OWASP ISVS Alignment**:
- V4.1.1: Use of secure communication protocols
- V4.1.2: Cryptographic verification of server certificates
- V4.2.1: Use of strong cipher suites

**OWASP ISTG Testing**:
- ISTG-FW-CRYPT-002: Test TLS configuration strength
- ISTG-DES-COMM-001: Verify secure communication implementation

**OWASP FSTM**:
- Stage 6: Runtime analysis of TLS implementation
- Stage 7: Network communication security testing

**OWASP IoTGoat**:
- Practice intercepting and analyzing device TLS traffic
- Test certificate validation implementations

### Resources

- [NIST PQC Standards](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [OQS OpenSSL Provider](https://github.com/open-quantum-safe/oqs-provider)
- [wolfSSL PQC](https://www.wolfssl.com/post-quantum-cryptography/)
- [Cloudflare PQC Research](https://blog.cloudflare.com/post-quantum-for-all/)
- [IETF PQC TLS Drafts](https://datatracker.ietf.org/doc/draft-ietf-tls-hybrid-design/)

**Other Example\(s\):**

To utilize TLS, there are other options besides OpenSSL. A non-exhaustive list is below.

Formerly PolarSSL, a list of projects using mbed TLS can be found at:

* [https://tls.mbed.org/kb/generic/projects-using-mbedtls](https://tls.mbed.org/kb/generic/projects-using-mbedtls)
* [https://tls.mbed.org/](https://tls.mbed.org/)

Examples of implementation can be found at

* [https://tls.mbed.org/kb/how-to/mbedtls-tutorial](https://tls.mbed.org/kb/how-to/mbedtls-tutorial)

Formerly CyaSSL, wolfSSL and a list of projects using wolfSSL can be found at:

* [https://www.wolfssl.com/wolfSSL/wolfssl-embedded-ssl-case-studies.html](https://www.wolfssl.com/wolfSSL/wolfssl-embedded-ssl-case-studies.html)
* [https://www.wolfssl.com/wolfSSL/Home.html](https://www.wolfssl.com/wolfSSL/Home.html)

Examples of implementation can be found at:

* [https://github.com/wolfSSL/wolfssl-examples](https://github.com/wolfSSL/wolfssl-examples)

## Additional References <a id="additional-references"></a>

### Certificate Authorities and Management
* [Let's Encrypt](https://letsencrypt.org/) - Free, automated certificate authority
* [Let's Encrypt for Embedded Devices](https://community.letsencrypt.org/t/certificate-for-embedded-device-without-a-domain-name/2372)
* [Smallstep - Certificates for IoT](https://smallstep.com/) - Private CA for embedded systems

### TLS Testing and Validation Tools
* [testssl.sh](https://testssl.sh/) - TLS/SSL testing tool
* [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) - Online TLS configuration analyzer
* [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) - Generate secure TLS configs
* [TLS Cipher Suite Search](https://ciphersuite.info/) - Cipher suite reference

### Standards and Guidelines
* [NIST SP 800-52 Rev. 2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-52r2.pdf) - Guidelines for TLS Implementations
* [RFC 8446 - TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446) - TLS 1.3 specification
* [Mozilla TLS Guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS) - Server-side TLS configuration
* [PCI DSS TLS Requirements](https://www.pcisecuritystandards.org/documents/Migrating_from_SSL_Early_TLS_Information%20Supplement_v1_1.pdf)

### Post-Quantum Cryptography
* [NIST Post-Quantum Cryptography](https://csrc.nist.gov/projects/post-quantum-cryptography) - PQC standardization
* [Open Quantum Safe](https://openquantumsafe.org/) - Open-source PQC implementations
* [Cloudflare PQC](https://blog.cloudflare.com/post-quantum-for-all/) - Post-quantum TLS implementations

### Hardware Security
* [ARM TrustZone](https://www.arm.com/technologies/trustzone-for-cortex-m) - Trusted Execution Environment
* [wolfSSL with TPM Support](https://www.wolfssl.com/products/wolfcrypt-and-tpm/) - Hardware-backed key storage
* [PKCS#11](https://docs.oasis-open.org/pkcs11/pkcs11-base/v3.0/pkcs11-base-v3.0.html) - Cryptographic Token Interface Standard

### Historical References and Case Studies
* [OpenSSL Cookbook](http://fm4dd.com/openssl/) - OpenSSL configuration examples
* [Wink Hub Certificate Expiration Incident](https://www.engadget.com/2015/04/19/wink-home-automation-hub-bricked/) - Importance of certificate management

### OWASP Resources
* [OWASP Transport Layer Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
* [OWASP ISVS V4 - Communication Requirements](https://github.com/OWASP/IoT-Security-Verification-Standard-ISVS/blob/master/en/V4-Communication_Requirements.md)
* [OWASP ISTG - Cryptography Testing](https://owasp.org/owasp-istg/03_test_cases/firmware/istg-fw-crypt.html)

