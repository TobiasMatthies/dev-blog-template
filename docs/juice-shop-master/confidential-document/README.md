---
id: confidential-document
slug: /juice-shop-master/confidential-document
title: Confidential Document Challenge
---

# Confidential Document Challenge

**Category:** Security Misconfiguration (OWASP Top 10: A05:2021 – Security Misconfiguration)

A walkthrough of the **Confidential Document** challenge in [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — finding a sensitive file hidden in an exposed FTP directory using directory enumeration.

## Video Walkthrough

<!-- TODO: add link to the video walkthrough (max. 5 min) -->

## Goal

Locate and access a confidential document that is publicly reachable but not linked anywhere in the application.

## Step 1: Discover the FTP Directory

While browsing the application, I opened the **Terms of Use** link and noticed that the URL pointed to a file served directly from an `/ftp/` path. This revealed that the FTP directory is publicly accessible and serving files without authentication — a clear misconfiguration worth investigating further.

## Step 2: Enumerate the FTP Directory with Gobuster

The `/ftp/` endpoint is accessible, so I ran Gobuster to discover hidden paths:

```bash
gobuster dir \
  -u http://127.0.0.1:3000/ftp/ \
  -w DirBuster-2007_directory-list-2.3-small.txt \
  -t 4 \
  -o results.txt \
  --exclude-length 2055
```

The `--exclude-length 2055` flag filters out the default 403 error body, leaving only genuine hits.

**Output:**

```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://127.0.0.1:3000/ftp/
[+] Method:                  GET
[+] Threads:                 4
[+] Wordlist:                DirBuster-2007_directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] Exclude Length:          2055
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
quarantine             (Status: 200) [Size: 9588]
Progress: 87662 / 87662 (100.00%)
===============================================================
Finished
===============================================================
```

## Step 3: Review the Results

```bash
cat results.txt
```

```
quarantine (Status: 200) [Size: 9588]
```

Only one subdirectory was found: `quarantine`.

## Step 4: Navigate to the File

Browsing to `http://127.0.0.1:3000/ftp/quarantine` revealed the quarantine folder. Going one level up to `/ftp/` exposed the **`acquisitions.md`** file — the confidential document that completes the challenge.

## Key Takeaways

- Exposed FTP directories without authentication are a common misconfiguration.
- Directory enumeration with a focused wordlist and response-size filtering keeps results clean and actionable.
- Always check parent directories after discovering a subdirectory — the target may not be in the deepest path.

## Security Risk & Impact

Serving a directory of files without authentication or an index/allow-list relies entirely on "security through obscurity" — the assumption that nobody will guess or enumerate the path. As shown here, that assumption breaks down against basic tooling like Gobuster in minutes. In a real environment this class of misconfiguration can expose confidential business documents, credentials, backup files, or source code to anyone who finds (or brute-forces) the URL, with no audit trail and no access control to stop it. The broader consequence is data exposure that can lead to further compromise (leaked secrets enabling account takeover, business intelligence leaking to competitors, etc.). The fix is always to enforce authentication/authorization at the storage layer itself, not to rely on a path being "hard to guess".
