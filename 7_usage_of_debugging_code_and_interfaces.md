# Usage of Debugging Code and Interfaces

It is important to ensure all unnecessary pre-production build code, as well as dead and unused code, has been removed prior to firmware release to all market segments. This includes but is not limited to potential backdoor code and root privilege accounts that may have been left by parties such as Original Design Manufacturers \(ODM\) and Third-Party contractors. Typically this falls in scope for Original Equipment Manufacturers \(OEM\) to perform via reverse engineering of binaries. This should also require ODMs to sign Master Service Agreements \(MSA\) insuring that either no backdoor code is included and that all code has been reviewed for software security vulnerabilities holding all Third-Party developers accountable for devices that are mass deployed into the market.

**Considerations \(Disclaimer: The List below is non-exhaustive\):**

* Remove backdoor accounts used for debugging, deployment verification and/or customer support purposes.
* Ensure third party libraries and binary images are reviewed for backdoors by staff before market deployment.
  * Tools such as Binwalk, Firmadyne, IDA pro, radare2, Firmware Mod Toolkit \(FMK\), Firmware Analysis Comparison Toolkit \(FACT\), and various other tools listed in the additional references \(\#4\) should be utilized for firmware analysis.

## Additional References <a id="additional-references"></a>

* [https://www.owasp.org/index.php/Leftover\_Debug\_Code](https://www.owasp.org/index.php/Leftover_Debug_Code)
* [https://cwe.mitre.org/data/definitions/489.html](https://cwe.mitre.org/data/definitions/489.html)
* [http://www.kb.cert.org/vuls/id/419568](http://www.kb.cert.org/vuls/id/419568)
* [Embedded Device Firmware Analysis Tools](https://www.owasp.org/index.php/OWASP_Embedded_Application_Security#tab=Embedded_Device_Firmware_Analysis_Tools)

