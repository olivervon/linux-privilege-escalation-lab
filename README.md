# Five86-1 Linux Privilege Escalation Lab

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
10. [Key Findings Summary](#10-key-findings-summary)
11. [Lessons Learned](#11-lessons-learned)
12. [Conclusion](#12-conclusion)

## 1. Overview

**Target:** Five86-1 / TwoThreeOne  
**Target IP:** `10.0.0.231`  
**Operating system:** Debian GNU/Linux 10  
**Assessment type:** Authorized lab penetration test  
**Focus areas:** Initial access, credential discovery, lateral movement, Linux privilege escalation, and evidence collection

### Objective

The objective was to gain initial access to the target, move between local user accounts, escalate privileges to root, and locate the flags placed throughout the lab.

The assessment focused on identifying misconfigurations and weak security controls that could allow a low-privileged user to gain elevated access.

## 2. Executive Summary

The assessment identified multiple weaknesses that could be chained together to obtain root-level access.

Initial access was obtained through a command execution vulnerability in OpenNetAdmin. Subsequent enumeration revealed credential material, an exposed password hash, an email containing another user's password, and an overly permissive `sudo` rule.

The final privilege escalation was achieved through a root-owned SUID executable located in the `moss` user's home directory. Running the executable resulted in a shell with an effective user ID of `0`.

Three flags were located during the lab. Their values are intentionally excluded from this report.

## 3. Attack Chain

| Stage | Account or context | Result |
|---|---|---|
| 1. Initial access | `www-data` | Obtained command execution through OpenNetAdmin |
| 2. Credential discovery | `douglas` | Recovered a password from an exposed password hash and gained SSH access |
| 3. Lateral movement | `jen` | Abused an overly permissive `sudo` rule to modify SSH authorized keys |
| 4. Lateral movement | `moss` | Discovered a password in a local email |
| 5. Privilege escalation | `root` | Exploited the behavior of a root-owned SUID executable |
| 6. Flag discovery | Root shell | Located the root flag and the remaining lab flag |

## 4. Initial Access: OpenNetAdmin

### Discovery

The target exposed the following services:

- TCP/22: OpenSSH
- TCP/80: Apache HTTP Server
- TCP/10000: Webmin

The web server hosted an OpenNetAdmin installation at `/ona/`. The identified application version was OpenNetAdmin 18.1.1.

### Exploitation

A publicly available exploit was used against the lab's OpenNetAdmin installation to obtain command execution as `www-data`.

The exploit script used was:

`/usr/share/exploitdb/exploits/php/webapps/47691.sh`

The command used to launch the exploit was:

```bash
bash /usr/share/exploitdb/exploits/php/webapps/47691.sh http://10.0.0.231/ona/
```

### Finding

**Severity: High**

The application allowed an attacker to execute commands on the host. This provided the initial foothold needed for further enumeration and privilege escalation.

### Remediation

- Upgrade OpenNetAdmin to a supported, patched version.
- Remove unused or exposed administrative applications.
- Restrict access to management interfaces.
- Monitor web application logs for suspicious requests and command execution.

## 5. Credential Discovery and Access as Douglas

Database configuration files were inspected during enumeration.

The following file contained database connection information:

`/opt/ona/www/local/config/database_settings.inc.php`

The configuration exposed credentials for the OpenNetAdmin database. Additional credential material was found in:

`/var/www/.htpasswd`

The file contained an Apache password hash associated with the `douglas` account.

### Password recovery

The hash was copied into a local file on the attacking machine and tested using John the Ripper with a constrained character mask based on the available lab clues.

The recovered password was then used to establish an SSH session as `douglas`.

```bash
ssh douglas@10.0.0.231
```

### Finding

**Severity: High**

Exposed credential material and a recoverable password allowed access to a local user account.

### Remediation

- Remove secrets from web-accessible directories and application configuration.
- Restrict access to password files.
- Use strong, unique passwords resistant to offline cracking.
- Rotate credentials after exposure.
- Avoid reusing credentials across services and accounts.

## 6. Lateral Movement: Douglas to Jen

### Discovery

The following command was used to enumerate the available `sudo` permissions:

```bash
sudo -l
```

The output showed that `douglas` could execute `/bin/cp` as `jen` without entering a password.

This permission was overly broad because it allowed the caller to choose the source and destination of a file copy with the privileges of another user.

### Exploitation

A new SSH key pair was generated, and the public key was added to a temporary file alongside the existing authorized key.

The key pair was generated with:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jen_lab -N ''
```

The existing authorized keys were copied into a temporary file, and the new public key was appended.

The temporary file was then copied to Jen's SSH authorization file using the permitted `sudo` command:

```bash
sudo -u jen /bin/cp /tmp/jen_keys /home/jen/.ssh/authorized_keys
```

SSH access as `jen` was subsequently obtained using the new private key.

```bash
ssh -i ~/.ssh/jen_lab jen@10.0.0.231
```

The account transition was verified:

```bash
id
```

Output confirmed UID `1003` and the `jen` group.

### Finding

**Severity: High**

The unrestricted use of `cp` as another user allowed modification of that user's SSH authorization configuration.

### Remediation

- Avoid granting unrestricted access to general-purpose file utilities through `sudo`.
- Replace broad command permissions with narrowly scoped administrative scripts.
- Protect SSH authorization files and monitor changes to them.
- Review `sudoers` rules for commands that permit arbitrary source and destination paths.

## 7. Lateral Movement: Jen to Moss

### Discovery

The user's mail file was inspected:

```bash
cat /var/mail/jen
```

An email from Roy contained Moss's newly assigned password and explicitly asked Jen to communicate it to Moss.

The email therefore provided a direct credential disclosure path.

### Exploitation

The password from the email was used to switch to Moss's account:

```bash
su - moss
```

The resulting identity was verified:

```bash
id
```

The output confirmed UID `1001` and the `moss` group.

### Finding

**Severity: High**

A plaintext password was disclosed through local email. Any account able to read that message could use the credential to access Moss's account.

### Remediation

- Do not transmit passwords in plaintext email.
- Use a secure credential-management process.
- Require unique credentials and rotate temporary passwords.
- Restrict access to local mailboxes and audit sensitive communications.

## 8. Privilege Escalation: Moss to Root

### Discovery

A running process named `upyourgame` was identified:

```bash
ps -ef | grep '[u]pyourgame'
```

The process was running as root.

The executable was located at:

`/home/moss/.games/upyourgame`

Its permissions were inspected:

```bash
ls -l /home/moss/.games/upyourgame
```

The output showed that the executable was owned by root and had the SUID bit enabled:

```text
-rwsr-xr-x 1 root root 16824 Jan 1 2020 /home/moss/.games/upyourgame
```

The file type was confirmed:

```bash
file /home/moss/.games/upyourgame
```

The output identified it as a 64-bit ELF executable with the set-user-ID bit enabled.

### Exploitation

The program was executed as Moss:

```bash
/home/moss/.games/upyourgame
```

The interactive program eventually exposed a shell. The resulting shell identity was verified with:

```bash
id
```

The output included:

```text
uid=0(root) gid=1001(moss) groups=1001(moss)
```

The effective UID of `0` confirmed root-level privileges.

### Finding

**Severity: Critical**

The root-owned SUID executable exposed a path to a root shell. A low-privileged user could execute the program and obtain root-level access.

### Remediation

- Remove the SUID bit unless it is strictly required.
- Audit all SUID and SGID executables.
- Review custom binaries for unsafe shell invocation, input handling, and privilege transitions.
- Avoid executing shell commands from privileged programs without strict input validation.
- Monitor for unexpected processes running with elevated effective privileges.

## 9. Flag Discovery and Evidence

Three flags were located during the lab. Their values are intentionally omitted.

One flag was found at:

`/var/www/html/reports/flag.txt`

After obtaining root-level access, another flag was located at:

`/root/flag.txt`

The remaining flag was found during subsequent inspection of files associated with the Roy account. The exact flag value and submission status are not included in this report.

For a reproducible portfolio record, retain screenshots or terminal output showing the discovery process while redacting flag contents, passwords, and private keys.

## 10. Key Findings Summary

| Finding | Impact | Recommended priority |
|---|---|---|
| Vulnerable OpenNetAdmin installation | Remote command execution | Critical |
| Exposed credential material | Account compromise | High |
| Recoverable user password hash | Unauthorized SSH access | High |
| Unrestricted `sudo` permission for `cp` | Lateral movement to another user | High |
| Plaintext password in local email | Credential disclosure | High |
| Root-owned SUID executable exposing a shell | Root compromise | Critical |

## 11. Lessons Learned

This lab demonstrated how individually weak controls can combine into a complete compromise.

Key lessons:

1. **Initial access is only the beginning.** After obtaining a foothold, enumerate local files, services, users, credentials, and permissions.
2. **Credentials are security boundaries.** Configuration files, password hashes, and local mail can reveal paths to additional accounts.
3. **Review `sudo` permissions carefully.** General-purpose commands can become privilege-escalation primitives when run as another user.
4. **SUID binaries require careful review.** A privileged executable must not expose unsafe behavior to unprivileged users.
5. **Verify privilege changes.** The `id` command provided clear evidence that the shell had an effective UID of `0`.
6. **Document evidence as you go.** Record commands, relevant outputs, account transitions, and remediation recommendations without publishing secrets.

## 12. Conclusion

The assessment progressed from remote command execution to root-level access through a chain of application, credential-management, `sudo`, and SUID weaknesses.

The most important defensive takeaway is that privilege escalation often results from several weaknesses working together rather than a single isolated vulnerability.

Fixing the initial application vulnerability, removing exposed credentials, narrowing `sudo` permissions, and auditing privileged binaries would substantially reduce the attack surface demonstrated in this lab.

---

**Portfolio note:** This report describes an authorized lab environment. All commands and observations refer to the lab target, not a production system.
