### Securing Sensitive Information {#4-securing-sensitive-information}

Do not hardcode secrets such as passwords, usernames, tokens, private keys or similar variants into firmware release images. This also includes the storage of sensitive data that is written to disk. If hardware security element \(SE\) or Trusted Execution Environment \(TEE\) is available, it is recommended to utilize such features for storing sensitive data. Otherwise, use of strong cryptography should be evaluated to protect the data.

If possible, all sensitive data in clear-text should be ephemeral by nature and reside in a volatile memory only.

**Noncompliant** [**Hardcoded Password**](https://www.owasp.org/index.php/Use_of_hard-coded_password) **Example:**

```c
int VerifyAdmin(char *password) {

  if (strcmp(password, "Mew!")) {
    printf("Incorrect Password!\n");
    return 0;
  }

  printf("Entering Diagnostic Mode\n");
  return 1;
}
```

**Noncompliant** [**Storing sensitive data to disk**](https://www.securecoding.cert.org/confluence/display/c/MEM06-C.+Ensure+that+sensitive+data+is+not+written+out+to+disk) **Example:**

In this noncompliant code example, sensitive information is supposedly stored in the dynamically allocated buffer, secret, which is processed and eventually cleared by a call to `memset_s()`. The memory page containing secret can be swapped out to disk. If the program crashes before the call to `memset_s()` completes, the information stored in secret may be stored in the core dump.

```c
char *secret;

secret = (char *)malloc(size+1);
if (!secret) {
  /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret, '\0', size+1);
free(secret);
secret = NULL;
```

To prevent the information from being written to a core dump, the size of core dumps that the program will generate should be set to 0 using `setrlimit()`:

```c
#include <sys/resource.h>
/* ... */
struct rlimit limit;
limit.rlim_cur = 0;
limit.rlim_max = 0;
if (setrlimit(RLIMIT_CORE, &limit) != 0) {
    /* Handle error */
}

char *secret;

secret = (char *)malloc(size+1);
if (!secret) {
  /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret, '\0', size+1);
free(secret);
secret = NULL;
```

Alternatively, the use of `mlock()` can be used to prevent paging by locking memory in place. This compliant solution not only disables the creation of core files but also ensures that the buffer is not swapped to hard disk:

```c
#include <sys/resource.h>
/* ... */
struct rlimit limit;
limit.rlim_cur = 0;
limit.rlim_max = 0;
if (setrlimit(RLIMIT_CORE, &limit) != 0) {
    /* Handle error */
}

long pagesize = sysconf(_SC_PAGESIZE);
if (pagesize == -1) {
  /* Handle error */
}

char *secret_buf;
char *secret;

secret_buf = (char *)malloc(size+1+pagesize);
if (!secret_buf) {
  /* Handle error */
}

/* mlock() may require that address be a multiple of PAGESIZE */
secret = (char *)((((intptr_t)secret_buf + pagesize - 1) / pagesize) * pagesize);

if (mlock(secret, size+1) != 0) {
    /* Handle error */
}

/* Perform operations using secret... */

memset_s(secret_buf, '\0', size+1+pagesize);
if (munlock(secret, size+1) != 0) {
    /* Handle error */
}
secret = NULL;

free(secret_buf);
secret_buf = NULL;
```

**Storing Sensitive Data, Noncompliant** [**Example**](https://www.securecoding.cert.org/confluence/display/c/MEM03-C.+Clear+sensitive+information+stored+in+reusable+resources): In this example, sensitive information stored in the dynamically allocated memory referenced by secret is copied to the dynamically allocated buffer, `new_secret`, which is processed and eventually deallocated by a call to `free()`. Because the memory is not cleared, it may be reallocated to another section of the program where the information stored in `new_secret` may be unintentionally leaked.

```c
char *secret;

/* Initialize secret */

char *new_secret;
size_t size = strlen(secret);
if (size == SIZE_MAX) {
  /* Handle error */
}

new_secret = (char *)malloc(size+1);
if (!new_secret) {
  /* Handle error */
}
strcpy(new_secret, secret);

/* Process new_secret... */

free(new_secret);
new_secret = NULL;
```

**Storing Sensitive Data, Compliant Example**: To prevent information leakage, dynamic memory containing sensitive information should be sanitized before being freed. Sanitization is commonly accomplished by clearing the allocated space \(that is, filling the space with '\0' characters\).

```c
char *secret;

/* Initialize secret */

char *new_secret;
size_t size = strlen(secret);
if (size == SIZE_MAX) {
  /* Handle error */
}

/* Use calloc() to zero-out allocated space */
new_secret = (char *)calloc(size+1, sizeof(char));
if (!new_secret) {
  /* Handle error */
}
strcpy(new_secret, secret);

/* Process new_secret... */

/* Sanitize memory  */
memset_s(new_secret, '\0', size);
free(new_secret);
new_secret = NULL;
```

**Considerations:**

* Do not hardcode certificates across product lines.
* Do not hardcode passwords across product lines.
* Do not store secrets in an unprotected storage location or external storage including within an EEPROM or flash.

#### Additional References {#additional-references}

* [https://cwe.mitre.org/data/definitions/259.html](https://cwe.mitre.org/data/definitions/259.html)
* [https://cwe.mitre.org/data/definitions/798.html](https://cwe.mitre.org/data/definitions/798.html)
* [https://cwe.mitre.org/data/definitions/321.html](https://cwe.mitre.org/data/definitions/321.html)
* [CWE-321: Use of Hard-coded Cryptographic Key - CVE-2013-6952](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2013-6952)
  * [http://www.ioactive.com/pdfs/IOActive\_Belkin-advisory-lite.pdf](http://www.ioactive.com/pdfs/IOActive_Belkin-advisory-lite.pdf)



