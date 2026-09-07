---
id: overview
slug: /juice-shop-master
title: Juice Shop Master
---

# Juice Shop Master

This project documents four hacking challenges solved against [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/), an intentionally insecure web application used for security training. Each challenge was investigated using Burp Suite and, where useful, automated with small scripts, and is documented with the exact steps taken to reproduce it. All content in this documentation is intended **for educational purposes only** and must only be used against systems you own or are explicitly authorized to test, such as a local Juice Shop instance.

## Table of Contents

| Challenge             | Category                  | Documentation                                                        | Video                         |
| --------------------- | ------------------------- | -------------------------------------------------------------------- | ----------------------------- |
| CAPTCHA Bypass        | Improper Input Validation | [captcha-bypass/README.md](./captcha-bypass/README.md)               | [Watch](https://www.loom.com/share/73e5edfe311d4fe7aee8f19259330a61) |
| Confidential Document | Security Misconfiguration | [confidential-document/README.md](./confidential-document/README.md) | <!-- TODO: add video link --> |
| API-Only XSS          | Injection (XSS)           | [api-only-xss/README.md](./api-only-xss/README.md)                   | [Watch](https://www.loom.com/share/18282a802ceb4a4dab6406be0e9e44bb) |
| Admin Registration    | Broken Access Control     | [admin-registration/README.md](./admin-registration/README.md)       | [Watch](https://www.loom.com/share/3973522537fc48e99433f9df879cf1d6) |

## Quickstart

1. Run a local OWASP Juice Shop instance, e.g. via Docker:
   ```bash
   docker run --rm -p 3000:3000 bkimminich/juice-shop
   ```
2. Open `http://127.0.0.1:3000` in your browser and set up an intercepting proxy (e.g. [Burp Suite](https://portswigger.net/burp)) to inspect requests.
3. Open the challenge folder you're interested in (see the table above) and follow the documented steps to reproduce the attack.
4. Watch the linked video for a narrated walkthrough of the same steps.

## Security Notes

- No real personal data, passwords, tokens, IP addresses, or SSH keys are stored anywhere in this repository; example values shown in requests/scripts are placeholders or have been redacted.
- All challenges were solved against a local instance of OWASP Juice Shop under `127.0.0.1`.
