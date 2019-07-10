# Injection Prevention

Ensure all untrusted data and user input is validated, sanitized, and/or output encoded to prevent unintended system execution. There are various injection attacks within application security such as operating system \(OS\) command injection, cross-site scripting \(E.g. JavaScript injection\), SQL injection, and others such as XPath injection. However, the most prevalent of the injection attacks within embedded software pertain to OS command injection; when an application accepts untrusted/insecure input and passes it to external applications \(either as the application name itself or arguments\) without validation or proper escaping.

[**Noncompliant Code Example**](https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=2130132) of using operating system calls:

In this noncompliant code example, the system\(\) function is used to execute any\_cmd in the host environment. Invocation of a command processor is not required.

```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

enum { BUFFERSIZE = 512 };

void func(const char *input) {
  char cmdbuf[BUFFERSIZE];
  int len_wanted = snprintf(cmdbuf, BUFFERSIZE,
                            "any_cmd '%s'", input);
  if (len_wanted >= BUFFERSIZE) {
    /* Handle error */
  } else if (len_wanted < 0) {
    /* Handle error */
  } else if (system(cmdbuf) == -1) {
    /* Handle error */
  }
}
```

If this code is compiled and run with elevated privileges on a Linux system, an attacker can create an account by entering the following string: `any_cmd ‘happy'; useradd 'attacker’` which would be interpreted as:

```c
any_cmd 'happy';
useradd 'attacker'
```

**Compliant Example**:In this compliant solution, the call to `system()` is replaced with a call to `execve()`. The exec family of functions does not use a full shell interpreter, so it is not vulnerable to command-injection attacks, such as the one illustrated in the noncompliant code example.

The `execlp()`, `execvp()`, and \(nonstandard\) `execvP()` functions duplicate the actions of the shell in searching for an executable file if the specified filename does not contain a forward slash character \(/\). As a result, they should be used without a forward slash character \(/\) only if the PATH environment variable is set to a safe value.

The `execl()`, `execle()`, `execv()`, and `execve()` functions do not perform path name substitution.

Additionally, precautions should be taken to ensure the external executable cannot be modified by an untrusted user, for example, by ensuring the executable is not writable by the user. This compliant solution is significantly different from the preceding noncompliant code example. First, input is incorporated into the args array and passed as an argument to `execve()`, eliminating concerns about buffer overflow or string truncation while forming the command string. Second, this compliant solution forks a new process before executing `/usr/bin/any_cmd` in the child process. Although this method is more complicated than calling system\(\), the added security is worth the additional effort.

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <errno.h>
#include <stdlib.h>

void func(char *input) {
  pid_t pid;
  int status;
  pid_t ret;
  char *const args[3] = {"any_exe", input, NULL};
  char **env;
  extern char **environ;

  /* ... Sanitize arguments ... */

  pid = fork();
  if (pid == -1) {
    /* Handle error */
  } else if (pid != 0) {
    while ((ret = waitpid(pid, &status, 0)) == -1) {
      if (errno != EINTR) {
        /* Handle error */
        break;
      }
    }
    if ((ret != -1) &&
      (!WIFEXITED(status) || !WEXITSTATUS(status)) ) {
      /* Report unexpected child status */
    }
  } else {
    /* ... Initialize env as a sanitized copy of environ ... */
    if (execve("/usr/bin/any_cmd", args, env) == -1) {
      /* Handle error */
      _Exit(127);
    }
  }
}
```

**Considerations:**

* Do not invoke shell command wrappers such as but not limited to:  
  * PHP:`system()` `exec()`
  * C:`system()`
  * C++:`ShellExecute()` 
  * Lua:`os.execute()`
  * Perl:`system()` `exec()`
  * Python:`os.system()` `subprocess.call()`
* If possible, avoid utilizing user data into operation system commands.
  * If needed, utilize lookup maps of numbers-to-command-strings for user driven strings that may be passed to the operating system.
* Whitelist accepted commands via a lookup map to ensure only expected parameter values are processed.
* Ensure to contextually output encode characters user data \(e.g. HTML, JavaScript, CSS, etc.\)
  * HTML Entity Encoding
    * &lt; output encoded to:
      * ```text
        &lt;
        ```
    * &gt; output encoded to:
      * `&gt;`

## Additional References: <a id="additional-references"></a>

* [Multiple Netgear routers are vulnerable to arbitrary command injection](https://www.kb.cert.org/vuls/id/582384)
* [FTC Charges D-Link Put Consumers’ Privacy at Risk Due to the Inadequate Security of Its Computer Routers and Cameras](https://www.ftc.gov/news-events/press-releases/2017/01/ftc-charges-d-link-put-consumers-privacy-risk-due-inadequate)
* [https://www.owasp.org/index.php/XSS\_\(Cross\_Site\_Scripting\)\_Prevention\_Cheat\_Sheet](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [https://www.owasp.org/index.php/SQL\_Injection\_Prevention\_Cheat\_Sheet](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)
* [https://www.owasp.org/images/2/2e/OWASP\_Code\_Review\_Guide-V1\_1.pdf](https://www.owasp.org/images/2/2e/OWASP_Code_Review_Guide-V1_1.pdf) \(Page 117\)
* [https://www.owasp.org/index.php/Command\_Injection](https://www.owasp.org/index.php/Command_Injection)
* [http://cwe.mitre.org/data/definitions/77.html](http://cwe.mitre.org/data/definitions/77.html)
* [http://cwe.mitre.org/data/definitions/78.html](http://cwe.mitre.org/data/definitions/78.html)
* [Bash Command Injection Vulnerability](https://ics-cert.us-cert.gov/advisories/ICSA-14-269-01A)

