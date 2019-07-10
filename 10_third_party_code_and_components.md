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
* Buildroot \(free\)
  * `make legal-info`
* Package Managers \(free\)
* * `dpkg --list`
  * `rpm -qa`
  * `yum list`
  * `apt list --installed`
* RetireJS for Javascript projects \(free\)

**A sample BOM is shown below:**

| **Component** | Version | Vulnerabilities - CVEs | Notes |
| :--- | :--- | :--- | :--- |
| jQuery | 1.4.4 | CVE-2011-4969 |  |
| libxml2 | 2.9.4 | CVE-2016-5131 | To be fixed |

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

**Utilizing LibScanner Example:**

Download the latest NVD xml DB

```bash
# ./download_xml.sh

...
...
--2017-02-20 14:57:57--  https://nvd.nist.gov/download/nvdcve-2017.xml.gz
Resolving nvd.nist.gov (nvd.nist.gov)... 129.6.13.177, 2610:20:6005:13::177
Connecting to nvd.nist.gov (nvd.nist.gov)|129.6.13.177|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 68023 (66K) [application/x-gzip]
Saving to: ‘nvdcve-2017.xml.gz’

nvdcve-2017.xml.gz  100%[===================>]  66.43K   389KB/s    in 0.2s…
```

Find your installed-packages.txt from your yocto build. For information on that see: [http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html\#understanding-what-the-build-history-contains](http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#understanding-what-the-build-history-contains)

For a visual representation of the discovered CVEs paste the content of installed-packages.txt to [http://devicevulnerabilitychecker.com](http://devicevulnerabilitychecker.com) to integrate it as part of your CI system, see below.

Run the scanner on your installed-packages.txt

```bash
# ./cli.py  --format yocto "path/to/installed-packages.txt" dbs/  > cve_test.xml
```

cve\_test will now include a list of 'unit tests' in XUnit format that fail for every cve not ignored

```bash
# tail cve_test.xml

<failure> Medium (6.8) - Use-after-free vulnerability in libxml2 through 2.9.4, as used in Google Chrome before 52.0.2743.82, allows remote attackers to cause a denial of service or possibly have unspecified other impact via vectors related to the XPointer range-to function. 

 CVE Published on: 2016-07-23 https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-5131 </failure>
</testcase>
<testcase id="CVE-2016-9318" name="CVE-2016-9318" classname="libxml2 - 2.9.4" time="0">
<failure> Medium (6.8) - libxml2 2.9.4 and earlier, as used in XMLSec 1.2.23 and earlier and other products, does not offer a flag directly indicating that the current document may be read but other files may not be opened, which makes it easier for remote attackers to conduct XML External Entity (XXE) attacks via a crafted document. 

 CVE Published on: 2016-11-15 https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-9318 </failure>
</testcase>
</testsuite>
```

**As of Yocto 2.2 Morty, a built-in** `cve-check` ****[**BitBake class**](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/classes/cve-check.bbclass) **was added to help automate checking of recipes against public CVEs.**

**TODO**

**Considerations \(Disclaimer: The List below is non-exhaustive\):**

* Use of [retire.js](https://github.com/RetireJS/retire.js) for JavaScript Libraries
  * Utilize `npm audit` for NodeJS packages
* Use [OWASP DependencyCheck](https://github.com/jeremylong/DependencyCheck) for detecting publicly disclosed vulnerabilities in application [dependencies and file types](https://jeremylong.github.io/DependencyCheck/analyzers/index.html).
* Use [`safety check`](https://github.com/pyupio/safety) for scanning python related packages for known vulnerabilities
* Use of [OWASP ZAP](https://github.com/zaproxy/zaproxy/wiki/Downloads) for web application testing
* Utilize tools such as [Lynis](https://raw.githubusercontent.com/CISOfy/lynis/master/lynis) for basic Kernel hardening auditing and suggestions.
  * `wget --no-check-certificate  https://github.com/CISOfy/lynis/archive/master.zip && unzip master.zip && cd lynis-master/ && bash lynis audit system`
  * Review the report in: `/var/log/lynis.log`
  * **Note**: Lynis will bypass Kernel checks if a Linux kernel is not in use. The following error message will be in the logs: “Skipped test KRNL-5695 \(Determine Linux kernel version and release number\) Reason to skip: Incorrect guest OS \(Linux only\)”
  * Lynis should be modified accordingly if storage is limited \(i.e. removing unnecessary plugins such as php etc.\)
* Utilize free library scanners such as [LibScanner](https://github.com/scriptingxss/LibScanner) which searches through a project's dependencies and cross references them with the NVD looking for known CVEs for a yocto build environment.
  * This tool outputs XML which enables teams to utilize such features for continuous integration testing.
* Utilize package managers \(opkg, ipkg, etc.. \) or custom update mechanisms for misc libraries within the toolchain.
* Review changelogs of toolchains, software packages, and libraries to better determine if an update is needed.
* Ensure the implementation of embedded build systems such as Yocto and Buildroot are set up in a way that allows for the update of all included packages.

## Additional References <a id="additional-references"></a>

* [https://www.kb.cert.org/vuls/id/922681](https://www.kb.cert.org/vuls/id/922681)
* [https://www.kb.cert.org/vuls/id/561444](https://www.kb.cert.org/vuls/id/561444)
* [https://buildroot.org/downloads/manual/manual.html\#faq-no-binary-packages](https://buildroot.org/downloads/manual/manual.html#faq-no-binary-packages)
* [https://wiki.yoctoproject.org/wiki/Security](https://wiki.yoctoproject.org/wiki/Security)
* [https://nvd.nist.gov/](https://nvd.nist.gov/)
* [https://www.openhub.net/](https://www.openhub.net/)
* [Improving Your Embedded Linux Security Posture with Yocto](https://legacy.gitbook.com/book/scriptingxss/embedded-appsec-best-practices/edit#)

