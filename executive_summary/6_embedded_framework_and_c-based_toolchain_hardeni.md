### 6. Embedded Framework and C-Based Toolchain Hardening {#6-embedded-framework-and-c-based-toolchain-hardening}

Limit BusyBox, embedded frameworks, and toolchains to only those libraries and functions being used when configuring firmware builds. Embedded Linux build systems such as Buildroot, Yocto and others typically perform this task. Removal of known insecure libraries and protocols such as Telnet not only minimize attack entry points in firmware builds, but also provide a secure-by-design approach to building software in efforts to thwart potential security threats.

**Hardening a library** [**Example**](https://www.owasp.org/index.php/C-Based_Toolchain_Hardening)**:** It is known that [compression is insecure](http://arstechnica.com/security/2012/09/crime-hijacks-https-sessions/) \(amongst others\),[SSLv2 is insecure](http://www.schneier.com/paper-ssl-revised.pdf), [SSLv3 is insecure](http://www.yaksman.org/~lweith/ssl.pdf), as well as early versions of TLS . In addition, suppose you don't use hardware and engines, and only allow static linking. Given the knowledge and specifications, you would configure the OpenSSL library as follows:

```bash
$ Configure darwin64-x86_64-cc -no-hw -no-engine -no-comp -no-shared -no-dso -no-ssl2 -no-ssl3 --openssldir=
```

**Selecting one shell Example**: Utilizing buildroot, the screenshot below demonstrates only one Shell being enabled, bash. \(_Note: Buildroot examples are shown below but there are other ways to accomplish the same configuration with other embedded Linux build systems._\)  
![](/assets/embedSec3.png)

**Hardening Services Example**: The screenshot below shows openssh enabled but not FTP daemons proftpd and pure-ftpd. Only enable FTP if TLS is to be utilized. For example, proftpd and pureftpd require custom compilation to use TLS with mod\_tls for proftpd and passing `./configure --with-tls` for pureftpd.![](/assets/embedSec2.png)

**Considerations \(Disclaimer: The List below is non-exhaustive\):**

* Ensure services such as SSH have a secure password created.
* Remove unused language interpreters such as: perl, python, lua.
* Remove dead code from unused library functions.
* Remove unused shell interpreters such as: ash, dash, zsh.
  * Review `/etc/shell`
* Remove legacy insecure daemons which includes but not limited to: Telnet, FTP, TFTP.
* Utilize tools such as [Lynis](https://raw.githubusercontent.com/CISOfy/lynis/master/lynis) for hardening auditing and suggestions.

  ```
  *   wget --no-check-certificate https://github.com/CISOfy/lynis/archive/master.zip && unzip master.zip && cd lynis-master/ && bash lynis audit system
  ```

  * Review the report in: `/var/log/lynis.log`

* Perform iterative threat model exercises with developers as well as relative stakeholders on software running on the embedded device.

#### Additional References {#additional-references}

* [https://www.owasp.org/index.php/C-Based\_Toolchain\_Hardening](https://www.owasp.org/index.php/C-Based_Toolchain_Hardening)
* [https://www.bulkorder.ftc.gov/system/files/publications/pdf0199-carefulconnections-buildingsecurityinternetofthings.pdf](https://www.bulkorder.ftc.gov/system/files/publications/pdf0199-carefulconnections-buildingsecurityinternetofthings.pdf)
* [http://isa99.isa.org/Public/Documents/ISA-62443-4-1-WD.pdf](http://isa99.isa.org/Public/Documents/ISA-62443-4-1-WD.pdf) \(page 34-38\)
* [https://events.linuxfoundation.org/sites/events/files/slides/belloni-petazzoni-buildroot-oe\_0.pdf](https://events.linuxfoundation.org/sites/events/files/slides/belloni-petazzoni-buildroot-oe_0.pdf) - Details on buildroot and yocto
* [http://elinux.org/Toolchains](http://elinux.org/Toolchains)
* [https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS](https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS)
* [http://www.proftpd.org/docs/howto/TLS.html](http://www.proftpd.org/docs/howto/TLS.html)
* [https://www.owasp.org/index.php/Application\_Threat\_Modeling](https://www.owasp.org/index.php/Application_Threat_Modeling)
* [GNU C Library Vulnerability in Industrial Products](http://www.siemens.com/cert/pool/cert/siemens_security_advisory_ssa-301706.pdf)
* [Linux Exploit Quick Listing](http://www.kmbl.us/les/working.php)



