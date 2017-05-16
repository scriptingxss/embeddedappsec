### **1. Buffer and Stack Overflow Protection** {#1-buffer-and-stack-overflow-protection}

Prevent the use of known dangerous functions and APIs in effort to protect against memory-corruption vulnerabilities within firmware. \(e.g. Use of [unsafe C functions](https://www.securecoding.cert.org/confluence/display/c/VOID+MSC34-C.+Do+not+use+deprecated+and+obsolete+functions) - [strcat, strcpy, sprintf, scanf](http://cwe.mitre.org/data/definitions/676.html#Demonstrative%20Examples\)\) Memory-corruption vulnerabilities, such as buffer overflows, can consist of overflowing the stack \([Stack overflow](https://en.wikipedia.org/wiki/Stack_buffer_overflow\)\) or overflowing the heap \([Heap overflow](https://en.wikipedia.org/wiki/Heap_overflow\)\). For simplicity purposes, this document does not distinguish between these two types of vulnerabilities. In the event a buffer overflow has been detected and exploited by an attacker, the instruction pointer register is overwritten to execute the arbitrary malicious code provided by the attacker.

**Finding Vulnerable C functions in source code. Example: Utilize the “find” command below within a “C” repository to find vulnerable C functions such as “strncpy” and “strlen” in source code.**

```
find . -type f -name '.c' -print0\|xargs -0 grep -e 'strncpy.strlen'\|wc -l
```
Usage of deprecated functions, [**Noncompliant Code Example**](https://www.securecoding.cert.org/confluence/display/c/VOID+STR35-C.+Do+not+copy+data+from+an+unbounded+source+to+a+fixed-length+array):This noncompliant code example assumes that gets\(\) will not read more than BUFSIZ - 1 characters from stdin. This is an invalid assumption, and the resulting operation can cause a buffer overflow. Note further that BUFSIZ is a macro integer constant, defined in stdio.h, representing a suggested argument to setvbuf\(\) and not the maximum size of such an input buffer.

The gets\(\) function reads characters from the stdin into a destination array until end-of-file is encountered or a newline character is read. Any newline character is discarded, and a null character is written immediately after the last character read into the array.
```c
#include <stdio.h>
 
void func(void) {
  char buf[BUFSIZ];
  if (gets(buf) == NULL) {
    /* Handle error */
  }
}
```
**Compliant Example**: The fgets\(\) function reads, at most, one less than a specified number of characters from a stream into an array. This solution is compliant because the number of bytes copied from stdin to buf cannot exceed the allocated memory:
```c
#include <stdio.h>
#include <string.h>
 
enum { BUFFERSIZE = 32 };
 
void func(void) {
  char buf[BUFFERSIZE];
  int ch;
 
  if (fgets(buf, sizeof(buf), stdin)) {
    /* fgets succeeds; scan for newline character */
    char *p = strchr(buf, '\n');
    if (p) {
      *p = '\0';
    } else {
      /* Newline not found; flush stdin to end of line */
      while (((ch = getchar()) != '\n')
            && !feof(stdin)
            && !ferror(stdin))
        ;
    }
  } else {
    /* fgets failed; handle error */
  }
}
```
strncat\(\) is a variation on the original strcat\(\) library function. Both are used to append one NULL terminated C string to another. The danger with the original strcat\(\) was that the caller might provide more data than can fit into the receiving buffer, thereby overrunning it. The most common result of this is a segmentation violation. A worse result is the silent and undetected corruption of whatever followed the receiving buffer in memory.

strncat\(\) adds an additional parameter allowing the user to specify the maximum number of bytes to copy. This is NOT the amount of data to copy. It is NOT the size of the source data. It is a limit to the amount of data to copy and is usually set to the size of the receiving buffer.

**Compliant Example usage of “strncat”:**
```c
char buffer[SOME_SIZE];

strncat( buffer, SOME_DATA, sizeof(buffer)-1);
```

**NonCompliant Example usage of “strncat**”**:**
```c
strncat( buffer, SOME_DATA, strlen( SOME_DATA )); 
```
The screenshot below demonstrates stack protection support being enabled while building a firmware image utilizing buildroot.
![](/assets/embedSec1.png)

#### 

**Considerations:**

* What kind of buffer and where it resides: physical, logical, virtual memory?
* What data will remain when the buffer is freed or left around to LRU out?
* What strategy will be followed to ensure old buffers do not leak data \(example: clear buffer after use\)?
* Initialize buffers to known value on allocation.
* Consider where variables are stored: stack, static or allocated structure.
* Dispose and securely wipe sensitive information stored in buffers or temporary files during runtime after they are no longer needed \(e.g. Wipe buffers from locations where personally identifiable information\(PII\) is stored before releasing the buffers\).
* Explicitly initialize variables.
* Ensure secure compiler flags or switches are utilized upon each firmware build. \(e.g. For GCC -fPIE, -fstack-protector-all, -Wl,-z,noexecstack, -Wl,-z,noexecheap etc.. See additional references section for more details.\)
* Use safe equivalent functions for known vulnerable functions such as \(non-exhaustive list below\):
  * `gets() -> fgets()`
  * `strcpy() -> strncpy()`
  * `strcat() -> strncat()`
  * `sprintf() -> snprintf()`
* Those functions that do not have safe equivalents should be rewritten with safe checks implemented.

#### Additional References {#additional-references}

* OSS \(Open Source Software\) Static Analysis Tools
  * Use of [flawfinder](http://www.dwheeler.com/flawfinder/) and [PMD](https://pmd.github.io/) for C
  * Use of [cppcheck](http://cppcheck.sourceforge.net/) for [C++](https://github.com/struct/mms/blob/master/Modern_Memory_Safety_In_C_CPP.pdf)
* [http://www.dwheeler.com/secure-programs/Secure-Programs-HOWTO/library-c.html](http://www.dwheeler.com/secure-programs/Secure-Programs-HOWTO/library-c.html)
* [https://www.owasp.org/index.php/C-Based\_Toolchain\_Hardening\#GCC.2FBinutils](https://www.owasp.org/index.php/C-Based_Toolchain_Hardening#GCC.2FBinutils)
* [https://www.owasp.org/index.php/Buffer\_overflow\_attack](https://www.owasp.org/index.php/Buffer_overflow_attack)
* [https://www.owasp.org/images/2/2e/OWASP\_Code\_Review\_Guide-V1\_1.pdf](https://www.owasp.org/images/2/2e/OWASP_Code_Review_Guide-V1_1.pdf) \(Page 113-114\)
* [University of Pittsburgh - Secure Coding C/C++: String Vulnerabilities \(PDF\)](http://www.sis.pitt.edu/jjoshi/courses/IS2620/Spring07/Lecture3.pdf)
* [Intel Open Source Technology Center SDL Banned Functions](https://github.com/01org/safestringlib/wiki/SDL-List-of-Banned-Functions)



