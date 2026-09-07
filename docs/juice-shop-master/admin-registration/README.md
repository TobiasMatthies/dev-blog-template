---
id: admin-registration
slug: /juice-shop-master/admin-registration
title: Admin Registration Challenge
---

# Admin Registration Challenge

**Category:** Broken Access Control – Mass Assignment (OWASP Top 10: A01:2021 – Broken Access Control)

A walkthrough of the **Admin Registration** challenge in [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — registering a new account with administrator privileges by exploiting a mass-assignment flaw in the registration endpoint.

## Video Walkthrough (german)

https://www.loom.com/share/3973522537fc48e99433f9df879cf1d6

## Goal

Register a new user account that has administrator rights.

## Step 1: Inspect a Normal Registration with Burp Suite

I signed up a new customer account through the regular registration form while intercepting the traffic with Burp Suite. Comparing the request and the server's response revealed that the response body includes a `"role": "customer"` field on the newly created user — meaning the server tracks a `role` per account and is willing to tell the client what it is.

## Step 2: Test Whether the Endpoint Accepts a `role` Field

I sent the original registration request to Burp Repeater and added a `role` field to the `POST` payload, setting it to `"administrator"`:

```
POST /api/Users/ HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/json

{
  "email": "test@test.com",
  "password": "...",
  "role": "administrator"
}
```

This returned a validation error stating that the value is not a valid option for the field. That error is itself the key finding: it confirms the endpoint does inspect and process the `role` field from client input — it just rejected this particular value. In other words, the API performs mass assignment on a field that should never be client-controlled in the first place.

## Step 3: Brute-Force the Correct Role Value

Since the field is processed but the exact accepted value/spelling was unknown, I built a small list of likely synonyms for "administrator" (`admin`, `Admin`, `ADMIN`, `administrator`, `root`, `superuser`, ...) and ran each one through the payload using Burp Repeater, replacing the `role` value on every request.

The value `"admin"` was accepted by the server and returned a `201 Created` response — the account was registered with administrator rights.

## Step 4: Verify Admin Access

Logging in with the newly created account and accessing an admin-only area (e.g. the administration section) confirmed the elevated privileges, completing the challenge.

## Key Takeaways

- A field that is only _displayed_ to the client (like `role` in the registration response) may still be _accepted_ as writable input by the same endpoint — always test round-tripping fields you only expected to be read-only.
- A validation error naming the field and listing it as "not a valid option" is a strong signal that the field is processed server-side and just needs the right value, not that it's ignored.
- Client-controlled fields that affect authorization (roles, permissions, ownership) must be explicitly excluded from mass-assignment/binding on any endpoint that accepts user input, especially unauthenticated ones like registration.

## Security Risk & Impact

This is a textbook mass-assignment vulnerability: the registration endpoint binds the entire incoming JSON body to the user model instead of allow-listing only the fields a new, unauthenticated user should be able to set (email, password, etc.). Because privilege-related fields like `role` are not excluded, anyone can self-register as an administrator without ever needing valid admin credentials or exploiting any authentication weakness. The impact of this class of vulnerability is severe and immediate: full compromise of the application's access control model, letting an attacker view/modify all user data, manage products and orders, and access any other admin-only functionality — effectively a complete takeover with a single unauthenticated HTTP request. The fix is to always construct the persisted object from an explicit allow-list of client-writable fields, never from the raw request body, especially on any endpoint that isn't itself behind an authorization check.
