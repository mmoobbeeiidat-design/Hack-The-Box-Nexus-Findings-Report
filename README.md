🛡️ Hack The Box - Nexus Findings Report

📋 Overview
This repository contains a comprehensive penetration test report for the Hack The Box machine Nexus.

🔍 Key Findings

Exposed Credentials in Gitea Commit History - Medium Severity
Cleartext DB_PASSWORD leaked in the commit history of a public krayin-docker-setup repository, granting access to the Krayin CRM application.

Krayin CRM Unrestricted File Upload (CVE-2026-38526) - High Severity
Authenticated file upload via the email attachment feature, bypassed with Burp Suite by renaming a PHP reverse shell from .png to .php, yielding a shell as www-data.

Gitea Template Sync Directory Traversal - Critical Severity
Root access obtained by abusing an unsanitized os.path.join() call in a systemd-triggered template-sync script, allowing an SSH public key to be planted directly into /root/.ssh/authorized_keys.

User flag captured: 3290dde5fca195ea451632d54d4b60da
Root flag captured: c25914b5b518500d1c71da5c8331ac9d

🛠️ Frameworks Used
PentNote - Automated documentation - https://github.com/A1GCH-afk/PentNote
MITRE ATT&CK - T1190, T1552.004, T1078.003, T1053.006, T1005
NSA D3FEND - D3-APIT, D3-FUAC, D3-CWAM

🧰 Tools & Scripts
build.py - Custom Python script that crafts raw Git tree/commit objects containing a directory-traversal payload, bypassing Git's own path validation to write an SSH public key outside the intended template-staging directory.
The script writes Git blob/tree/commit objects directly into .git/objects/, chaining five levels of ../ so that the synced path resolves to /root/.ssh/authorized_keys, then force-pushes the crafted commit so the target's template-sync service picks it up on its next scheduled run.

Download / usage:
# Download the script
wget https://raw.githubusercontent.com/mmoobbeeiidat-design/Hack-The-Box-Nexus-Findings-Report/main/build.py

# Generate an SSH key pair to plant
ssh-keygen -t ed25519 -f /tmp/.k -N ''

# Run inside a Gitea repository marked as a Template
python3 build.py

# Push the crafted commit to trigger the sync
git push -u origin main --force

The script expects a public key at /tmp/.k.pub and must be run from inside the root of the cloned template repository (where a .git directory is present).

php-reverse-shell.php - Third-party PHP reverse shell payload by pentestmonkey, used as the file-upload payload for initial access. Not included in this repository; download directly from the original source:
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php

📄 Full Report
View the complete report: Nexus.md
