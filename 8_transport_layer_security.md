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

## Post-Quantum Cryptography for TLS

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

## Embedded TLS Libraries: mbedTLS and wolfSSL

For resource-constrained embedded devices, specialized TLS libraries optimized for small footprint and low memory usage are essential alternatives to OpenSSL.

### mbedTLS (formerly PolarSSL)

**mbedTLS** is designed specifically for embedded systems with minimal resource requirements.

**Key Features**:
- Small footprint: ROM usage from 60KB (minimal) to 300KB (full)
- Low memory: 3.5KB RAM for basic TLS handshake
- Modular design: Enable only needed features
- Hardware acceleration support (AES-NI, ARMv8 Crypto Extensions)
- PSA Crypto API for secure key storage

#### mbedTLS Yocto Integration

**Add mbedTLS to Yocto build**:

```bitbake
# local.conf
IMAGE_INSTALL:append = " mbedtls"

# For development headers
IMAGE_INSTALL:append = " mbedtls-dev"
```

**mbedTLS Recipe Example**:

```bitbake
# recipes-connectivity/mbedtls-app/mbedtls-app_1.0.bb
DESCRIPTION = "Application using mbedTLS"
LICENSE = "MIT"

DEPENDS = "mbedtls"

SRC_URI = "file://tls-client.c"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o tls-client \
        ${WORKDIR}/tls-client.c \
        -lmbedtls -lmbedx509 -lmbedcrypto
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 tls-client ${D}${bindir}/
}
```

#### mbedTLS TLS 1.3 Client Example

```c
// mbedtls-tls13-client.c - TLS 1.3 client with mbedTLS
#include <mbedtls/net_sockets.h>
#include <mbedtls/ssl.h>
#include <mbedtls/entropy.h>
#include <mbedtls/ctr_drbg.h>
#include <mbedtls/error.h>
#include <string.h>
#include <stdio.h>

#define SERVER_NAME "api.example.com"
#define SERVER_PORT "443"
#define GET_REQUEST "GET / HTTP/1.1\r\nHost: api.example.com\r\n\r\n"

int main(void) {
    int ret;
    mbedtls_net_context server_fd;
    mbedtls_entropy_context entropy;
    mbedtls_ctr_drbg_context ctr_drbg;
    mbedtls_ssl_context ssl;
    mbedtls_ssl_config conf;
    mbedtls_x509_crt cacert;

    unsigned char buf[1024];
    const char *pers = "tls_client";

    // Initialize contexts
    mbedtls_net_init(&server_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_x509_crt_init(&cacert);
    mbedtls_ctr_drbg_init(&ctr_drbg);
    mbedtls_entropy_init(&entropy);

    // Seed random number generator
    ret = mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy,
                                 (const unsigned char *)pers, strlen(pers));
    if (ret != 0) {
        printf("mbedtls_ctr_drbg_seed failed: -0x%x\n", -ret);
        goto exit;
    }

    // Load CA certificates
    ret = mbedtls_x509_crt_parse_file(&cacert, "/etc/ssl/certs/ca-certificates.crt");
    if (ret < 0) {
        printf("mbedtls_x509_crt_parse failed: -0x%x\n", -ret);
        goto exit;
    }

    // Connect to server
    printf("Connecting to %s:%s...\n", SERVER_NAME, SERVER_PORT);
    ret = mbedtls_net_connect(&server_fd, SERVER_NAME, SERVER_PORT, MBEDTLS_NET_PROTO_TCP);
    if (ret != 0) {
        printf("mbedtls_net_connect failed: -0x%x\n", -ret);
        goto exit;
    }

    // Setup SSL/TLS configuration
    ret = mbedtls_ssl_config_defaults(&conf,
                                      MBEDTLS_SSL_IS_CLIENT,
                                      MBEDTLS_SSL_TRANSPORT_STREAM,
                                      MBEDTLS_SSL_PRESET_DEFAULT);
    if (ret != 0) {
        printf("mbedtls_ssl_config_defaults failed: -0x%x\n", -ret);
        goto exit;
    }

    // Configure TLS 1.3 only
    mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_3);
    mbedtls_ssl_conf_max_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_3);

    // Set cipher suites (TLS 1.3)
    static const int ciphersuites[] = {
        MBEDTLS_TLS1_3_AES_128_GCM_SHA256,
        MBEDTLS_TLS1_3_AES_256_GCM_SHA384,
        MBEDTLS_TLS1_3_CHACHA20_POLY1305_SHA256,
        0
    };
    mbedtls_ssl_conf_ciphersuites(&conf, ciphersuites);

    // Configure certificate verification
    mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
    mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
    mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);

    // Setup SSL context
    ret = mbedtls_ssl_setup(&ssl, &conf);
    if (ret != 0) {
        printf("mbedtls_ssl_setup failed: -0x%x\n", -ret);
        goto exit;
    }

    // Set hostname for SNI
    ret = mbedtls_ssl_set_hostname(&ssl, SERVER_NAME);
    if (ret != 0) {
        printf("mbedtls_ssl_set_hostname failed: -0x%x\n", -ret);
        goto exit;
    }

    mbedtls_ssl_set_bio(&ssl, &server_fd, mbedtls_net_send, mbedtls_net_recv, NULL);

    // Perform TLS handshake
    printf("Performing TLS handshake...\n");
    while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            printf("mbedtls_ssl_handshake failed: -0x%x\n", -ret);
            goto exit;
        }
    }

    // Verify peer certificate
    uint32_t flags = mbedtls_ssl_get_verify_result(&ssl);
    if (flags != 0) {
        char vrfy_buf[512];
        mbedtls_x509_crt_verify_info(vrfy_buf, sizeof(vrfy_buf), "  ! ", flags);
        printf("Certificate verification failed:\n%s\n", vrfy_buf);
        goto exit;
    }

    printf("TLS handshake successful\n");
    printf("Protocol: %s\n", mbedtls_ssl_get_version(&ssl));
    printf("Ciphersuite: %s\n", mbedtls_ssl_get_ciphersuite(&ssl));

    // Send HTTP request
    printf("Sending request...\n");
    while ((ret = mbedtls_ssl_write(&ssl, (unsigned char *)GET_REQUEST,
                                    strlen(GET_REQUEST))) <= 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            printf("mbedtls_ssl_write failed: -0x%x\n", -ret);
            goto exit;
        }
    }

    // Receive response
    printf("Reading response...\n");
    do {
        memset(buf, 0, sizeof(buf));
        ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf) - 1);

        if (ret == MBEDTLS_ERR_SSL_WANT_READ || ret == MBEDTLS_ERR_SSL_WANT_WRITE)
            continue;

        if (ret <= 0) {
            if (ret == MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY)
                break;

            printf("mbedtls_ssl_read failed: -0x%x\n", -ret);
            break;
        }

        printf("%s", buf);
    } while (1);

    // Close connection
    mbedtls_ssl_close_notify(&ssl);

exit:
    mbedtls_net_free(&server_fd);
    mbedtls_x509_crt_free(&cacert);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_ctr_drbg_free(&ctr_drbg);
    mbedtls_entropy_free(&entropy);

    return ret;
}
```

### wolfSSL - High-Performance Embedded TLS

**wolfSSL** provides a production-grade TLS stack optimized for embedded devices with extensive hardware support.

**Key Features**:
- Minimal footprint: 20-100KB depending on configuration
- FIPS 140-2/140-3 validated options
- Hardware crypto acceleration (Intel AES-NI, ARM Crypto, TPM)
- TLS 1.3 with 0-RTT support
- DTLS for IoT protocols

#### wolfSSL Yocto Integration

**Add wolfSSL to Yocto**:

```bitbake
# local.conf
IMAGE_INSTALL:append = " wolfssl"

# Enable specific features
PACKAGECONFIG:pn-wolfssl = "tls13 aesni harden"
```

**wolfSSL Recipe with Hardware Acceleration**:

```bitbake
# recipes-connectivity/wolfssl/wolfssl_%.bbappend
PACKAGECONFIG[tls13] = "--enable-tls13,--disable-tls13"
PACKAGECONFIG[aesni] = "--enable-aesni,--disable-aesni"
PACKAGECONFIG[harden] = "--enable-harden,--disable-harden"
PACKAGECONFIG[tpm] = "--enable-wolftpm,--disable-wolftpm,wolftpm"

# Production build flags
EXTRA_OECONF += "\
    --enable-fortress \
    --enable-secure-renegotiation \
    --disable-examples \
"

# Security hardening
CFLAGS:append = " -DWOLFSSL_ALWAYS_VERIFY_CB"
```

#### wolfSSL mTLS (Mutual TLS) Server Example

```c
// wolfssl-mtls-server.c - mTLS server for device authentication
#include <wolfssl/options.h>
#include <wolfssl/ssl.h>
#include <wolfssl/error-ssl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define SERVER_PORT 8443
#define CERT_FILE "/etc/ssl/server-cert.pem"
#define KEY_FILE "/etc/ssl/server-key.pem"
#define CA_FILE "/etc/ssl/client-ca.pem"

int main(void) {
    int sockfd, connd;
    struct sockaddr_in servAddr;
    struct sockaddr_in clientAddr;
    socklen_t size = sizeof(clientAddr);
    char buffer[256];
    int ret;

    WOLFSSL_CTX* ctx;
    WOLFSSL* ssl;

    // Initialize wolfSSL
    wolfSSL_Init();

    // Create SSL context
    ctx = wolfSSL_CTX_new(wolfTLSv1_3_server_method());
    if (ctx == NULL) {
        fprintf(stderr, "wolfSSL_CTX_new failed\n");
        return EXIT_FAILURE;
    }

    // Load server certificate
    if (wolfSSL_CTX_use_certificate_file(ctx, CERT_FILE, SSL_FILETYPE_PEM) != SSL_SUCCESS) {
        fprintf(stderr, "Failed to load server certificate\n");
        goto cleanup_ctx;
    }

    // Load server private key
    if (wolfSSL_CTX_use_PrivateKey_file(ctx, KEY_FILE, SSL_FILETYPE_PEM) != SSL_SUCCESS) {
        fprintf(stderr, "Failed to load server private key\n");
        goto cleanup_ctx;
    }

    // Load client CA certificate for mutual TLS
    if (wolfSSL_CTX_load_verify_locations(ctx, CA_FILE, NULL) != SSL_SUCCESS) {
        fprintf(stderr, "Failed to load client CA certificate\n");
        goto cleanup_ctx;
    }

    // Require client certificate (mutual TLS)
    wolfSSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);

    // Configure TLS 1.3 only
    wolfSSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION);

    // Set cipher suites
    if (wolfSSL_CTX_set_cipher_list(ctx, "TLS13-AES128-GCM-SHA256:TLS13-AES256-GCM-SHA384") != SSL_SUCCESS) {
        fprintf(stderr, "Failed to set cipher list\n");
        goto cleanup_ctx;
    }

    // Create socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        goto cleanup_ctx;
    }

    // Allow socket reuse
    int enable = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));

    // Bind socket
    memset(&servAddr, 0, sizeof(servAddr));
    servAddr.sin_family = AF_INET;
    servAddr.sin_port = htons(SERVER_PORT);
    servAddr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sockfd, (struct sockaddr*)&servAddr, sizeof(servAddr)) < 0) {
        perror("bind");
        goto cleanup_socket;
    }

    // Listen for connections
    if (listen(sockfd, 5) < 0) {
        perror("listen");
        goto cleanup_socket;
    }

    printf("mTLS server listening on port %d\n", SERVER_PORT);

    while (1) {
        // Accept connection
        connd = accept(sockfd, (struct sockaddr*)&clientAddr, &size);
        if (connd < 0) {
            perror("accept");
            continue;
        }

        printf("Connection accepted\n");

        // Create SSL object
        ssl = wolfSSL_new(ctx);
        if (ssl == NULL) {
            fprintf(stderr, "wolfSSL_new failed\n");
            close(connd);
            continue;
        }

        wolfSSL_set_fd(ssl, connd);

        // Perform TLS handshake
        ret = wolfSSL_accept(ssl);
        if (ret != SSL_SUCCESS) {
            int err = wolfSSL_get_error(ssl, ret);
            char errString[80];
            wolfSSL_ERR_error_string(err, errString);
            fprintf(stderr, "TLS handshake failed: %s\n", errString);
            wolfSSL_free(ssl);
            close(connd);
            continue;
        }

        printf("TLS handshake successful\n");

        // Get client certificate information
        WOLFSSL_X509* cert = wolfSSL_get_peer_certificate(ssl);
        if (cert) {
            char* subject = wolfSSL_X509_NAME_oneline(
                wolfSSL_X509_get_subject_name(cert), NULL, 0);
            char* issuer = wolfSSL_X509_NAME_oneline(
                wolfSSL_X509_get_issuer_name(cert), NULL, 0);

            printf("Client certificate:\n");
            printf("  Subject: %s\n", subject);
            printf("  Issuer: %s\n", issuer);

            XFREE(subject, NULL, DYNAMIC_TYPE_OPENSSL);
            XFREE(issuer, NULL, DYNAMIC_TYPE_OPENSSL);
            wolfSSL_X509_free(cert);
        }

        // Receive data
        memset(buffer, 0, sizeof(buffer));
        ret = wolfSSL_read(ssl, buffer, sizeof(buffer) - 1);
        if (ret > 0) {
            printf("Received: %s\n", buffer);

            // Send response
            const char* response = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK";
            wolfSSL_write(ssl, response, strlen(response));
        }

        // Cleanup
        wolfSSL_shutdown(ssl);
        wolfSSL_free(ssl);
        close(connd);
    }

cleanup_socket:
    close(sockfd);
cleanup_ctx:
    wolfSSL_CTX_free(ctx);
    wolfSSL_Cleanup();
    return EXIT_SUCCESS;
}
```

## Certificate Agility and Crypto-Agility

**Certificate agility** is the ability to quickly replace cryptographic algorithms, key sizes, and certificates in response to vulnerabilities or regulatory changes.

### The Need for Crypto-Agility

**Historical Context**:
- **MD5**: Broken in 2004, still found in legacy systems (2025)
- **SHA-1**: Deprecated in 2017, banned by browsers in 2020
- **RSA-1024**: Deprecated in 2010, still used in some IoT devices
- **3DES**: Deprecated in 2023, replaced by AES

**Future Threats**:
- Quantum computers will break RSA, ECDSA (2030-2040 timeframe)
- Algorithm vulnerabilities discovered unpredictably
- Regulatory requirements change (FIPS, NIST, industry-specific)

### Implementing Certificate Agility in Embedded Systems

#### Algorithm Negotiation Framework

```c
// crypto-agility.c - Algorithm negotiation framework
#include <stdio.h>
#include <string.h>

typedef enum {
    ALGO_RSA_2048,
    ALGO_RSA_3072,
    ALGO_RSA_4096,
    ALGO_ECDSA_P256,
    ALGO_ECDSA_P384,
    ALGO_ECDSA_P521,
    ALGO_ED25519,
    ALGO_MLDSA_44,      // Post-quantum
    ALGO_MLDSA_65,      // Post-quantum
    ALGO_MLDSA_87,      // Post-quantum
    ALGO_UNKNOWN
} crypto_algorithm_t;

typedef struct {
    crypto_algorithm_t algorithm;
    const char* name;
    int key_size;
    int deprecated;
    int year_deprecated;
} algorithm_info_t;

static const algorithm_info_t algorithm_table[] = {
    {ALGO_RSA_2048, "RSA-2048", 2048, 1, 2030},
    {ALGO_RSA_3072, "RSA-3072", 3072, 0, 0},
    {ALGO_RSA_4096, "RSA-4096", 4096, 0, 0},
    {ALGO_ECDSA_P256, "ECDSA-P256", 256, 0, 0},
    {ALGO_ECDSA_P384, "ECDSA-P384", 384, 0, 0},
    {ALGO_ECDSA_P521, "ECDSA-P521", 521, 0, 0},
    {ALGO_ED25519, "Ed25519", 256, 0, 0},
    {ALGO_MLDSA_44, "ML-DSA-44", 128, 0, 0},  // PQC
    {ALGO_MLDSA_65, "ML-DSA-65", 192, 0, 0},  // PQC
    {ALGO_MLDSA_87, "ML-DSA-87", 256, 0, 0},  // PQC
    {ALGO_UNKNOWN, NULL, 0, 0, 0}
};

crypto_algorithm_t negotiate_algorithm(crypto_algorithm_t* client_algos, int client_count,
                                       crypto_algorithm_t* server_algos, int server_count) {
    // Prefer strongest non-deprecated algorithm supported by both sides

    // Priority order: PQC > ECC > RSA
    crypto_algorithm_t priority_order[] = {
        ALGO_MLDSA_87,
        ALGO_MLDSA_65,
        ALGO_MLDSA_44,
        ALGO_ECDSA_P521,
        ALGO_ECDSA_P384,
        ALGO_ECDSA_P256,
        ALGO_ED25519,
        ALGO_RSA_4096,
        ALGO_RSA_3072,
        ALGO_RSA_2048
    };

    for (int i = 0; i < sizeof(priority_order) / sizeof(priority_order[0]); i++) {
        crypto_algorithm_t candidate = priority_order[i];

        // Check if deprecated
        const algorithm_info_t* info = &algorithm_table[candidate];
        if (info->deprecated) {
            continue;
        }

        // Check if supported by both client and server
        int client_supports = 0, server_supports = 0;

        for (int j = 0; j < client_count; j++) {
            if (client_algos[j] == candidate) {
                client_supports = 1;
                break;
            }
        }

        for (int j = 0; j < server_count; j++) {
            if (server_algos[j] == candidate) {
                server_supports = 1;
                break;
            }
        }

        if (client_supports && server_supports) {
            printf("Negotiated algorithm: %s\n", info->name);
            return candidate;
        }
    }

    return ALGO_UNKNOWN;
}

void check_algorithm_health(crypto_algorithm_t algo, int current_year) {
    const algorithm_info_t* info = &algorithm_table[algo];

    if (info->deprecated) {
        printf("⚠️  WARNING: %s is deprecated (since %d)\n",
               info->name, info->year_deprecated);

        if (current_year >= info->year_deprecated) {
            printf("❌ CRITICAL: %s should not be used in production\n", info->name);
        }
    } else {
        printf("✅ %s is currently approved for use\n", info->name);
    }
}
```

#### Automated Certificate Rotation

```bash
#!/bin/bash
# cert-rotation.sh - Automated certificate rotation for embedded devices

set -e

CERT_DIR="/etc/ssl/device"
NEW_CERT_DIR="/tmp/new-certs"
BACKUP_DIR="/var/backups/certs"
DEVICE_ID=$(cat /etc/device-id)
CA_SERVER="https://ca.example.com"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/cert-rotation.log
}

check_cert_expiry() {
    local cert_file=$1
    local warn_days=$2

    expiry_date=$(openssl x509 -in "$cert_file" -noout -enddate | cut -d= -f2)
    expiry_epoch=$(date -d "$expiry_date" +%s)
    current_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - current_epoch) / 86400 ))

    if [ $days_left -lt $warn_days ]; then
        log "Certificate expires in $days_left days (threshold: $warn_days)"
        return 1
    fi

    log "Certificate valid for $days_left days"
    return 0
}

request_new_certificate() {
    log "Requesting new certificate from CA..."

    # Generate new key pair
    openssl ecparam -genkey -name prime256v1 -out "$NEW_CERT_DIR/device-key.pem"

    # Generate CSR
    openssl req -new -key "$NEW_CERT_DIR/device-key.pem" \
        -out "$NEW_CERT_DIR/device.csr" \
        -subj "/CN=device-$DEVICE_ID/O=MyCompany/C=US"

    # Submit CSR to CA
    response=$(curl -s -X POST "$CA_SERVER/api/v1/certificate/request" \
        -H "Authorization: Bearer $(cat /etc/ca-token)" \
        -F "csr=@$NEW_CERT_DIR/device.csr" \
        -F "device_id=$DEVICE_ID")

    cert_id=$(echo "$response" | jq -r '.certificate_id')

    if [ "$cert_id" == "null" ]; then
        log "ERROR: Certificate request failed"
        return 1
    fi

    # Poll for certificate issuance
    for i in {1..30}; do
        sleep 2
        cert_status=$(curl -s "$CA_SERVER/api/v1/certificate/$cert_id/status" \
            -H "Authorization: Bearer $(cat /etc/ca-token)")

        status=$(echo "$cert_status" | jq -r '.status')

        if [ "$status" == "issued" ]; then
            # Download certificate
            curl -s "$CA_SERVER/api/v1/certificate/$cert_id/download" \
                -H "Authorization: Bearer $(cat /etc/ca-token)" \
                -o "$NEW_CERT_DIR/device-cert.pem"

            log "New certificate downloaded"
            return 0
        elif [ "$status" == "failed" ]; then
            log "ERROR: Certificate issuance failed"
            return 1
        fi
    done

    log "ERROR: Certificate issuance timeout"
    return 1
}

install_new_certificate() {
    log "Installing new certificate..."

    # Backup old certificates
    mkdir -p "$BACKUP_DIR/$(date +%Y%m%d-%H%M%S)"
    cp "$CERT_DIR"/* "$BACKUP_DIR/$(date +%Y%m%d-%H%M%S)/"

    # Install new certificate and key
    cp "$NEW_CERT_DIR/device-cert.pem" "$CERT_DIR/"
    cp "$NEW_CERT_DIR/device-key.pem" "$CERT_DIR/"
    chmod 600 "$CERT_DIR/device-key.pem"

    # Restart services using the certificate
    systemctl restart nginx
    systemctl restart mqtt-broker

    log "New certificate installed successfully"
}

rollback_certificate() {
    log "Rolling back to previous certificate..."

    latest_backup=$(ls -t "$BACKUP_DIR" | head -1)
    cp "$BACKUP_DIR/$latest_backup"/* "$CERT_DIR/"

    systemctl restart nginx
    systemctl restart mqtt-broker

    log "Certificate rollback complete"
}

main() {
    log "=== Certificate Rotation Started ==="

    mkdir -p "$NEW_CERT_DIR"

    # Check if renewal is needed (30 days before expiration)
    if check_cert_expiry "$CERT_DIR/device-cert.pem" 30; then
        log "Certificate renewal not needed yet"
        exit 0
    fi

    # Request new certificate
    if ! request_new_certificate; then
        log "ERROR: Failed to obtain new certificate"
        exit 1
    fi

    # Verify new certificate
    if ! openssl verify -CAfile /etc/ssl/ca-cert.pem "$NEW_CERT_DIR/device-cert.pem"; then
        log "ERROR: New certificate verification failed"
        exit 1
    fi

    # Install new certificate
    if ! install_new_certificate; then
        log "ERROR: Certificate installation failed"
        rollback_certificate
        exit 1
    fi

    # Test connectivity
    sleep 5
    if ! curl -s --cacert /etc/ssl/ca-cert.pem https://localhost:443/health > /dev/null; then
        log "ERROR: Health check failed after certificate installation"
        rollback_certificate
        exit 1
    fi

    log "=== Certificate Rotation Completed Successfully ==="
    rm -rf "$NEW_CERT_DIR"
}

main "$@"
```

**Cron Schedule for Automated Rotation**:

```cron
# /etc/cron.d/cert-rotation
# Check daily at 2 AM
0 2 * * * root /usr/local/bin/cert-rotation.sh
```

### TLS 1.3 Deployment Best Practices

**TLS 1.3 Benefits**:
- Faster handshake (1-RTT vs 2-RTT)
- 0-RTT resumption for returning clients
- Improved security (removed weak ciphers)
- Forward secrecy mandatory
- Encrypted handshake

**Configuration for Embedded Systems**:

```c
// tls13-config.c - TLS 1.3 configuration helper
#include <wolfssl/options.h>
#include <wolfssl/ssl.h>

WOLFSSL_CTX* create_secure_tls13_context(int is_server) {
    WOLFSSL_METHOD* method;
    WOLFSSL_CTX* ctx;

    wolfSSL_Init();

    // TLS 1.3 only
    method = is_server ? wolfTLSv1_3_server_method() : wolfTLSv1_3_client_method();
    ctx = wolfSSL_CTX_new(method);

    if (!ctx) {
        return NULL;
    }

    // Set minimum protocol version (TLS 1.3)
    wolfSSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION);
    wolfSSL_CTX_set_max_proto_version(ctx, TLS1_3_VERSION);

    // Configure cipher suites (TLS 1.3 only)
    // Priority: AEAD ciphers with perfect forward secrecy
    const char* ciphers =
        "TLS13-AES-256-GCM-SHA384:"
        "TLS13-CHACHA20-POLY1305-SHA256:"
        "TLS13-AES-128-GCM-SHA256";

    if (wolfSSL_CTX_set_cipher_list(ctx, ciphers) != SSL_SUCCESS) {
        wolfSSL_CTX_free(ctx);
        return NULL;
    }

    // Enable session tickets for 0-RTT (optional)
    wolfSSL_CTX_set_options(ctx, SSL_OP_NO_TICKET);  // Disable for security

    // Security hardening
    wolfSSL_CTX_set_options(ctx, SSL_OP_NO_COMPRESSION);
    wolfSSL_CTX_set_options(ctx, SSL_OP_NO_RENEGOTIATION);

    // OCSP stapling (client-side validation)
    wolfSSL_CTX_EnableOCSPStapling(ctx);

    return ctx;
}
```

### Hardware-Backed Key Storage Integration

**TPM 2.0 Integration with wolfSSL**:

```c
// tpm-wolfssl.c - Use TPM for private key operations
#include <wolfssl/options.h>
#include <wolfssl/ssl.h>
#include <wolftpm/tpm2.h>
#include <wolftpm/tpm2_wrap.h>

typedef struct {
    WOLFTPM2_DEV dev;
    WOLFTPM2_KEY tpmKey;
} TPM_CTX;

static TPM_CTX tpm_ctx;

int tpm_sign_callback(WOLFSSL* ssl, const byte* in, word32 inSz,
                      byte* out, word32* outSz, const byte* keyDer,
                      word32 keySz, void* ctx) {
    TPM_CTX* tpm = (TPM_CTX*)ctx;
    int rc;

    // Use TPM to sign
    rc = wolfTPM2_SignHash(&tpm->dev, &tpm->tpmKey, in, inSz, out, (int*)outSz);

    return rc == TPM_RC_SUCCESS ? 0 : -1;
}

WOLFSSL_CTX* create_tpm_context(void) {
    WOLFSSL_CTX* ctx;
    int rc;

    // Initialize TPM
    rc = wolfTPM2_Init(&tpm_ctx.dev, NULL, NULL);
    if (rc != TPM_RC_SUCCESS) {
        printf("TPM initialization failed\n");
        return NULL;
    }

    // Load TPM key (created separately)
    rc = wolfTPM2_ReadPublicKey(&tpm_ctx.dev, &tpm_ctx.tpmKey, TPM_20_ECC_KEY_HANDLE);
    if (rc != TPM_RC_SUCCESS) {
        printf("Failed to load TPM key\n");
        return NULL;
    }

    // Create SSL context
    ctx = wolfSSL_CTX_new(wolfTLSv1_3_client_method());
    if (!ctx) {
        return NULL;
    }

    // Register TPM signing callback
    wolfSSL_CTX_SetEccSignCb(ctx, tpm_sign_callback);
    wolfSSL_CTX_SetEccSignCtx(ctx, &tpm_ctx);

    return ctx;
}
```

### Historical References and Case Studies
* [OpenSSL Cookbook](http://fm4dd.com/openssl/) - OpenSSL configuration examples
* [Wink Hub Certificate Expiration Incident](https://www.engadget.com/2015/04/19/wink-home-automation-hub-bricked/) - Importance of certificate management

### OWASP Resources
* [OWASP Transport Layer Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
* [OWASP ISVS V4 - Communication Requirements](https://github.com/OWASP/IoT-Security-Verification-Standard-ISVS/blob/master/en/V4-Communication_Requirements.md)
* [OWASP ISTG - Cryptography Testing](https://owasp.org/owasp-istg/03_test_cases/firmware/istg-fw-crypt.html)

