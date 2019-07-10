# Transport Layer Security

Ensure all methods of communication are utilizing industry standard encryption configurations for [TLS](https://www.securecoding.cert.org/confluence/display/c/API10-C.+APIs+should+have+security+options+enabled+by+default). The use of TLS ensures that all data remains confidential and untampered with while in transit. Utilize free certificate authority services such as [Let’s Encrypt](https://letsencrypt.org/) if the embedded device utilizes domain names.

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

* Use the latest possible version of TLS for new products \(as of writing, this is TLS 1.2\)
* Consider implementing TLS two-way authentication for firmware that accepts TLS connections from a limited group of allowed clients.
* If possible, consider using mutual-authentication to authenticate both end-points.
* Validate the certificate public key, hostname, and [chain](http://fm4dd.com/openssl/certverify.htm).
* Ensure certificate and their chains use SHA256 for signing.
* Disable deprecated SSL and early TLS versions.
* Disable deprecated, NULL and weak cipher suites.
* Ensure private key and certificates are stored securely - e.g. Secure Environment or Trusted Execution Environment, or protected using strong cryptography.
* Keep certificates updated with up to date secure configurations.
* Ensure proper certificate update features are available upon expiration.
* Verify TLS configurations utilizing services such as [ssllabs.com](https://www.ssllabs.com), nmap using `--script ssl-enum-ciphers.nse`, TestSSLServer.jar, sslscan and sslyze.

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

* [https://letsencrypt.org/](https://letsencrypt.org/)
* [https://community.letsencrypt.org/t/certificate-for-embedded-device-without-a-domain-name/2372](https://community.letsencrypt.org/t/certificate-for-embedded-device-without-a-domain-name/2372)
* [http://fm4dd.com/openssl/](http://fm4dd.com/openssl/)
* [https://www.engadget.com/2015/04/19/wink-home-automation-hub-bricked/](https://www.engadget.com/2015/04/19/wink-home-automation-hub-bricked/)

