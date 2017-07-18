###  3. Firmware Updates and Cryptographic Signatures {#3-firmware-updates-and-cryptographic-signatures}

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
* Ensure updates are downloaded over the most recent secure TLS version possible. \(As of writing, this is TLS1.2\)
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

#### Additional References {#additional-references}

* [https://www.kernel.org/signature.html](https://www.kernel.org/signature.html)
* [https://www.owasp.org/index.php/Key\_Management\_Cheat\_Sheet](https://www.owasp.org/index.php/Key_Management_Cheat_Sheet)
* [http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r4.pdf](http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r4.pdf)
* [https://www.math.utah.edu/~beebe/PGP-notes.html](https://www.math.utah.edu/~beebe/PGP-notes.html)
* [CWE-321: Use of Hard-coded Cryptographic Key - CVE-2013-6952](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2013-6952)
  * [http://www.ioactive.com/pdfs/IOActive\_Belkin-advisory-lite.pdf](http://www.ioactive.com/pdfs/IOActive_Belkin-advisory-lite.pdf)
* [Implementing secure remote firmware updates](https://www.allegrosoft.com/wp-content/uploads/Secure-Firmware-Updates-Paper.pdf)
* [Code Integrity during execution by Automotive Grade Linux](http://docs.automotivelinux.org/docs/architecture/en/dev/reference/security/05-security-concepts.html#code-integrity-during-execution)
* [Securing Software Updates for Automobiles](https://uptane.github.io/)



