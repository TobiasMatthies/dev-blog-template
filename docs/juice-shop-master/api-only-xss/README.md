---
id: api-only-xss
slug: /juice-shop-master/api-only-xss
title: API-Only XSS Challenge
---

# API-Only XSS Challenge

**Category:** Injection – Cross-Site Scripting (OWASP Top 10: A03:2021 – Injection)

A walkthrough of the **API-only XSS** challenge in [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — injecting a persistent XSS payload directly through the product API, bypassing any client-side input restrictions in the UI.

## Video Walkthrough (german)

https://www.loom.com/share/18282a802ceb4a4dab6406be0e9e44bb

## Goal

Perform a persisted XSS attack via the product API without using the frontend admin form for it — i.e. the payload has to be delivered by talking to the API directly.

## Step 1: Enumerate the Product API with Burp Suite

I browsed the product pages of the shop while intercepting traffic with Burp Suite to map out which API endpoints are involved in loading and managing product data. This surfaced `api/Products/{id}`, which — besides serving product details via `GET` — also accepts `PUT` requests to update a product as a logged-in customer. This immediately stood out: an endpoint that lets a regular customer modify product data is worth a closer look, since product fields (like the description) are rendered elsewhere in the shop without necessarily being re-sanitized.

## Step 2: Retrieve the Existing Data Structure

Before crafting a malicious request, I sent a plain `GET` request to `api/Products/{id}` to capture the exact JSON structure the API expects, and copied the full response body as the basis for my `PUT` payload — this avoids missing any required fields and dropping/breaking product data.

## Step 3: Inject the XSS Payload

I modified the `description` field of the copied payload, replacing it with:

```html
<iframe src="javascript:alert(`xss`)"></iframe>
```

and sent this as the body of a `PUT` request to `api/Products/{id}` via Burp Repeater, authenticated as a logged-in customer.

```
PUT /api/Products/1 HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/json
Authorization: Bearer <token>

{
  "id": 1,
  "name": "...",
  "description": "<iframe src=\"javascript:alert(`xss`)\">",
  "price": ...,
  ...
}
```

The API accepted the update without sanitizing the injected markup.

## Step 4: Trigger the Payload

Navigating back to the product's detail page (where the `description` field is rendered) executed the injected `<iframe>` payload, popping the `alert('xss')` and completing the challenge — confirming that the description field is rendered unsanitized and that the write access to it is not restricted to admin users only.

## Key Takeaways

- API endpoints often enforce weaker (or different) input validation than the corresponding UI forms — testing the API directly can reveal injection points the frontend hides.
- An endpoint intended for admin-only product management should never accept writes from an authenticated customer role without an explicit authorization check.
- Any user-controllable field that is later rendered in HTML (product descriptions, comments, reviews, etc.) must be sanitized/escaped on output, regardless of which client or role wrote it.

## Security Risk & Impact

Persistent (stored) XSS is one of the most damaging classes of injection vulnerability because the payload is saved server-side and executed in the browser of every user who later views the affected page — here, every visitor to that product's detail page. An attacker exploiting this could steal session tokens/cookies, perform actions on behalf of victims (e.g. placing orders, changing account details), redirect users to phishing pages, or deploy browser-based malware, all without the victim doing anything more than viewing a normal product page. Because the injection happened through a supposedly internal/admin API rather than the public-facing form, it also highlights a broken access control problem underneath the XSS: input validation and authorization must be enforced consistently across every entry point into the data, not just the ones exposed in the UI.
