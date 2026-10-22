# Five86-1 Linux Privilege Escalation Lab

A documented security assessment of the Five86-1 Linux lab, covering initial access, credential discovery, lateral movement, privilege escalation, and remediation.

## Table of Contents

1. [Overview](#1-overview)
2. [Executive Summary](#2-executive-summary)
3. [Attack Chain](#3-attack-chain)
4. [Initial Access: OpenNetAdmin](#4-initial-access-opennetadmin)
5. [Credential Discovery and Access as Douglas](#5-credential-discovery-and-access-as-douglas)
6. [Lateral Movement: Douglas to Jen](#6-lateral-movement-douglas-to-jen)
7. [Lateral Movement: Jen to Moss](#7-lateral-movement-jen-to-moss)
8. [Privilege Escalation: Moss to Root](#8-privilege-escalation-moss-to-root)
9. [Flag Discovery and Evidence](#9-flag-discovery-and-evidence)
10. [Key Findings and Risk Assessment](#10-key-findings-and-risk-assessment)
11. [Remediation Priorities](#11-remediation-priorities)
12. [Lessons Learned](#12-lessons-learned)
13. [Conclusion](#13-conclusion)

---

## 1. Overview

| Field | Details |
|---|---|
| Target | Five86-1 |
| Hostname | TwoThreeOne |
| Target IP | `10.0.0.231` |
| Operating system | Debian GNU/Linux 10 |
| Assessment type | Authorized lab penetration test |
| Focus areas | Initial access, credential discovery, lateral movement, privilege escalation |

### Objective

The objective was to obtain initial access to the target, investigate its security configuration, move between local user accounts, and escalate privileges to root.

The assessment focused on identifying weaknesses that could be chained together to compromise the system.

This report documents the observed attack path, relevant technical evidence, security implications, and recommended remediation measures.

### Scope

The assessment was conducted against the Five86-1 lab target at `10.0.0.231`.

All activities described in this report relate to the authorized lab environment.

## 2. Executive Summary

The assessment demonstrated how multiple security weaknesses could be chained together to obtain root-level privileges on a Linux system.

Initial access was obtained through command execution in the OpenNetAdmin web application. Subsequent enumeration revealed exposed application configuration, a recoverable password hash, an overly permissive `sudo` rule, and a password disclosed through local email.

These weaknesses enabled movement between the `www-data`, `douglas`, `jen`, and `moss` accounts.

The final privilege escalation involved a root-owned SUID executable located in Moss's home directory. Executing the program exposed a shell with an effective user ID of `0`.

Three flags were reported as discovered during the lab. Their values are intentionally excluded from this report.

### Main findings

- Command execution through a vulnerable web application.
- Exposed application configuration and credential material.
- A recoverable password hash associated with a local user.
- An overly permissive `sudo` rule allowing file operations as another user.
- Plaintext credential disclosure through local email.
- A root-owned SUID executable that exposed a path to root-level privileges.

The combined weaknesses enabled a progression from initial access to root-level effective privileges.

## 3. Attack Chain

The following table summarizes the observed sequence.

| Stage | Account or context | Result |
|---|---|---|
| 1. Initial access | `www-data` | Obtained command execution through OpenNetAdmin |
| 2. Credential discovery | `douglas` | Recovered a password from an exposed hash and obtained SSH access |
| 3. Lateral movement | `jen` | Modified SSH authorization using a `sudo`-permitted file-copy operation |
| 4. Lateral movement | `moss` | Used a password disclosed in a local email |
| 5. Privilege escalation | Effective UID 0 | Obtained a shell with root-level effective privileges |
| 6. Flag discovery | Multiple locations | Located three lab flags |

The attack path illustrates how application vulnerabilities, credential exposure, excessive privileges, and unsafe privileged executables can combine into a complete system compromise.

## 4. Initial Access: OpenNetAdmin

### 4.1 Discovery

The target exposed the following services during reconnaissance.

| Port | Service |
|---|---|
| TCP/22 | OpenSSH |
| TCP/80 | Apache HTTP Server |
| TCP/10000 | Webmin |

An OpenNetAdmin installation was identified at `/ona/` on the web server. The identified application version was OpenNetAdmin 18.1.1.

### 4.2 Exploitation

A publicly available exploit was used against the lab installation to obtain command execution as `www-data`.

The exploit script used was:

`/usr/share/exploitdb/exploits/php/webapps/47691.sh`

The command used to launch the exploit was:

```bash
bash /usr/share/exploitdb/exploits/php/webapps/47691.sh http://10.0.0.231/ona/
```

The resulting access provided an initial foothold for further enumeration.

### 4.3 Security impact

**Observed result:** Command execution was obtained on the target as `www-data`.

This access allowed further investigation of application files, local configuration, and potential credential exposure.

**Preliminary severity: High.**

The final severity depends on the application's exposure, the privileges of the compromised service account, and the potential for subsequent privilege escalation.

### 4.4 Recommended remediation

- Upgrade or replace the vulnerable application with a supported, patched version.
- Remove applications that are no longer required.
- Restrict access to administrative interfaces.
- Run web applications with the minimum privileges necessary.
- Review web server logs for suspicious requests and command execution.
- Investigate whether application compromise could expose credentials or sensitive files.

## 5. Credential Discovery and Access as Douglas

### 5.1 Application configuration

During local enumeration, the following configuration file was identified:

`/opt/ona/www/local/config/database_settings.inc.php`

The file contained database connection information for OpenNetAdmin.

Exposed database credentials can permit unauthorized database access, depending on the account's permissions and whether the credentials are reused elsewhere.

The exact privileges and subsequent use of these database credentials were not established in this assessment.

### 5.2 Password hash discovery

Additional credential material was discovered in:

`/var/www/.htpasswd`

The file contained an Apache password hash associated with the `douglas` account.

The hash was copied to a local file on the attacking machine and tested using John the Ripper with a constrained character mask based on the available lab clues.

The password was recovered and subsequently used to establish SSH access.

```bash
ssh douglas@10.0.0.231
```

### 5.3 Security impact

**Observed result:** SSH access was obtained as `douglas` using a recovered password.

Exposed password hashes create an offline password-cracking risk. If a password is recovered, the associated account may be compromised.

**Preliminary severity: High.**

### 5.4 Recommended remediation

- Remove unnecessary secrets from application configuration files.
- Restrict access to credential stores and sensitive configuration.
- Use strong, unique passwords and appropriate password-hashing mechanisms.
- Rotate credentials after suspected exposure.
- Avoid password reuse across applications and system accounts.
- Restrict database accounts to the minimum permissions required.
- Review whether web service accounts can read files containing sensitive credentials.

## 6. Lateral Movement: Douglas to Jen

### 6.1 Discovery

The available `sudo` permissions were enumerated using:

```bash
sudo -l
```

The output showed that `douglas` could execute `/bin/cp` as `jen` without entering a password.

Allowing a user to execute a general-purpose file-copy utility as another user can be dangerous when the caller controls the source and destination paths.

In this case, the permission allowed modification of Jen's SSH authorization file.

### 6.2 Exploitation

An Ed25519 SSH key pair was generated for the lab:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jen_lab -N ''
```

The existing authorized keys were copied into a temporary file, and the newly generated public key was appended.

The temporary file was then copied to Jen's SSH authorization file using the permitted `sudo` command:

```bash
sudo -u jen /bin/cp /tmp/jen_keys /home/jen/.ssh/authorized_keys
```

SSH access was subsequently obtained as `jen` using the corresponding private key:

```bash
ssh -i ~/.ssh/jen_lab jen@10.0.0.231
```

The account transition was checked using:

```bash
id
```

The recorded output confirmed UID `1003` and the `jen` group.

### 6.3 Security impact

**Observed result:** Access was obtained as `jen` after modifying the account's SSH authorization configuration.

The `sudo` rule allowed a lower-privileged user to modify a file controlling authentication for another account.

**Preliminary severity: High.**

### 6.4 Recommended remediation

- Avoid unrestricted `sudo` access to general-purpose file utilities.
- Replace broad command permissions with narrowly scoped administrative scripts.
- Review `sudoers` rules for unsafe combinations of commands and arguments.
- Restrict write access to SSH authorization files.
- Monitor changes to `authorized_keys` files.
- Alert on unexpected SSH key additions and modifications to privileged users' home directories.

## 7. Lateral Movement: Jen to Moss

### 7.1 Discovery

The local mail file associated with Jen was inspected:

```bash
cat /var/mail/jen
```

An email from Roy contained Moss's newly assigned password and asked Jen to communicate it to Moss.

The message exposed a credential in plaintext to an account that could read the mailbox.

### 7.2 Exploitation

The password disclosed in the email was used to switch to Moss's account:

```bash
su - moss
```

The resulting identity was checked using:

```bash
id
```

The recorded output confirmed UID `1001` and the `moss` group.

### 7.3 Security impact

**Observed result:** Access was obtained as `moss` using a password disclosed through local email.

The exposure of authentication secrets through email can allow unauthorized account access and enable further movement through a system.

**Preliminary severity: High.**

### 7.4 Recommended remediation

- Do not transmit passwords in plaintext email.
- Use an approved secure process for distributing temporary credentials.
- Require unique credentials for each account.
- Rotate temporary credentials after initial use or suspected exposure.
- Restrict access to local mailboxes.
- Review mailbox permissions and audit sensitive communications.

## 8. Privilege Escalation: Moss to Root

### 8.1 Discovery

During process enumeration, a process named `upyourgame` was identified:

```bash
ps -ef | grep '[u]pyourgame'
```

The process was reported to be running as root.

The executable was located at:

`/home/moss/.games/upyourgame`

Its permissions were inspected using:

```bash
ls -l /home/moss/.games/upyourgame
```

The recorded output was:

```text
-rwsr-xr-x 1 root root 16824 Jan 1 2020 /home/moss/.games/upyourgame
```

The `s` in the owner's execute position indicates that the SUID bit was set.

The file type was also checked:

```bash
file /home/moss/.games/upyourgame
```

The output identified it as a 64-bit ELF executable with the set-user-ID bit enabled.

### 8.2 Exploitation

The executable was launched as Moss:

```bash
/home/moss/.games/upyourgame
```

The interactive program eventually exposed a shell.

The shell identity was checked using:

```bash
id
```

The recorded output was:

```text
uid=0(root) gid=1001(moss) groups=1001(moss)
```

The effective UID of `0` indicates root-level effective privileges. The retained group ID is a separate detail and does not invalidate the evidence of an effective UID of `0`.

### 8.3 Security impact

**Observed result:** The executable provided a shell with an effective UID of `0`.

A privileged executable that exposes a root-level shell to an unprivileged user represents a critical privilege-escalation risk.

**Severity: Critical.**

### 8.4 Recommended remediation

- Remove the SUID bit if the executable does not require it.
- Audit SUID and SGID executables across the system.
- Investigate why the executable was installed in a user's home directory.
- Review custom privileged binaries for unsafe shell invocation and privilege handling.
- Avoid executing shell commands from privileged programs unless the implementation is carefully secured.
- Monitor for unexpected SUID files and processes running with elevated effective privileges.

### 8.5 Verification

The recorded `id` output supports the conclusion that the shell had an effective UID of `0`.

When retaining evidence for the portfolio, ensure that the terminal screenshot matches the output quoted in this section.

## 9. Flag Discovery and Evidence

Three flags were reported as discovered during the lab. Their values are intentionally omitted.

Two recorded locations were:

- `/var/www/html/reports/flag.txt`
- `/root/flag.txt`

The remaining flag was found during subsequent inspection of files associated with the Roy account. Its exact location and submission status are not established in this report.

### Evidence handling

Retain relevant terminal output and screenshots demonstrating:

- The OpenNetAdmin installation and initial access.
- Credential discovery, with secrets redacted.
- The `sudo` permission and transition to Jen.
- The email-based credential disclosure and transition to Moss.
- The SUID executable's permissions and the resulting effective UID.
- Flag discovery without publishing the flag values.

Do not publish passwords, private SSH keys, recovered secrets, or actual flag values.

## 10. Key Findings and Risk Assessment

The following severity ratings are preliminary assessments based on the observed lab behavior. They are not the result of a formal CVSS calculation.

| Finding | Security impact | Preliminary severity |
|---|---|---|
| Vulnerable OpenNetAdmin installation | Remote command execution | High |
| Exposed application credential material | Potential unauthorized database access | High |
| Recoverable password hash | Unauthorized SSH access | High |
| Overly permissive `sudo` rule for `cp` | Modification of another user's SSH authorization file | High |
| Plaintext password in local email | Unauthorized account access | High |
| Root-owned SUID executable exposing a shell | Root-level privilege escalation | Critical |

Severity should be reassessed against the actual exposure, required access, affected privileges, and potential impact in the target environment.

## 11. Remediation Priorities

### Priority 1: Remove the path to root

- Investigate and remediate the SUID executable.
- Remove unnecessary SUID permissions.
- Audit privileged binaries and their behavior.
- Verify that unprivileged users cannot obtain a root-level shell through the executable.

### Priority 2: Restrict privilege delegation

- Remove unrestricted `sudo` permissions for general-purpose file utilities.
- Review the complete `sudoers` configuration.
- Use narrowly scoped commands and scripts for administrative tasks.
- Monitor changes to SSH authorization files.

### Priority 3: Secure credentials

- Rotate exposed passwords and database credentials.
- Remove secrets from unnecessary or accessible files.
- Restrict access to credential stores.
- Prohibit plaintext password sharing through email.
- Enforce unique, strong credentials.

### Priority 4: Remediate initial access

- Patch or replace the vulnerable OpenNetAdmin installation.
- Restrict access to administrative web applications.
- Review logs for suspicious activity.
- Investigate the extent of access available to the compromised web service account.

### Priority 5: Improve monitoring

- Monitor authentication failures and unusual successful logins.
- Alert on unexpected SSH key changes.
- Detect modifications to privileged files.
- Monitor unexpected SUID executables and processes with elevated effective privileges.

## 12. Lessons Learned

This lab demonstrated how multiple weaknesses can combine to produce a complete compromise.

1. **Initial access is only the beginning.** Local enumeration can reveal additional weaknesses after a foothold has been established.
2. **Credential exposure creates attack paths.** Application configuration, password hashes, and local email can reveal opportunities to access additional accounts.
3. **`sudo` rules require careful review.** General-purpose utilities can become privilege-escalation mechanisms when executed as another user.
4. **SUID executables require scrutiny.** A privileged program must not expose unsafe functionality to unprivileged users.
5. **Privilege changes must be verified.** Commands such as `id` provide evidence of the current identity and effective privileges.
6. **Evidence supports technical conclusions.** Commands, outputs, screenshots, and clear explanations make findings easier to validate.
7. **Remediation should address the full chain.** Fixing only the initial vulnerability may leave other privilege-escalation paths available.

## 13. Conclusion

The Five86-1 assessment progressed from remote command execution to root-level effective privileges through a chain of application, credential-management, privilege-delegation, and SUID-related weaknesses.

The main defensive takeaway is that system compromise often results from multiple weaknesses working together rather than from one isolated vulnerability.

Patching the vulnerable application, securing exposed credentials, restricting `sudo` permissions, protecting SSH authorization files, and auditing privileged executables would address the principal weaknesses demonstrated in this lab.

---

**Portfolio note:** This report documents an authorized lab exercise for educational purposes. The findings describe the observed lab environment and should not be interpreted as evidence of compromise on a production system.
