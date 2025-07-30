---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Weak Password Hashing Using MD5 in Ednovation's AIMath Web App"
date: 2025-07-28
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, authentication, cryptography, md5, ednovation, vulnerability, aimath, api]
permalink: /disclosures/aimath-weak-password-hashing/
---

## Summary

A cryptographic weakness was discovered in the **AIMath Web App**, a math learning platform for children operated by Ednovation. User passwords are stored using the obsolete and insecure **MD5 hash algorithm**, exposing them to trivial cracking via modern brute-force or rainbow table attacks.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

During an investigation into another vulnerability involving **unauthenticated API endpoints** (see [Public Data Exposure via Broken Auth in AIMath Web App](/disclosures/aimath-web-broken-auth/)), the JSON response containing student information was found to include a field labeled `password`. Upon analysis, it was determined that this value is simply an **MD5 hash of the user's plaintext password**.

This presents a serious cryptographic vulnerability and a violation of modern password handling standards.

## Affected Product

- **App**: AIMath Web App
- **Vendor**:  [Ednovation](https://ednovation.com)
- **Subdomain**: `https://aimath.ednoland.com/`
- **Component**: Backend user account storage / API responses
- **Version**: Latest production version prior to Aug 2022

## Vulnerability Details

- The API responses from endpoints such as `GET aimath.ednoland.com/classes/student/2` return user objects that include:
  - `username`
  - `email_account`
  - `mobile`
  - `password` (in hashed form)
  - `password_plaintext`
- The `password` field is an **MD5 hash of the plaintext password**.

This usage of MD5 poses a severe cryptographic risk:
- MD5 is a **fast, deterministic, non-salted** algorithm, making it ideal for attackers to crack with precomputed hash tables or GPUs.
- No salting or key stretching mechanisms (e.g., PBKDF2, bcrypt, scrypt) were used.

## Exploitation

1. The attacker accesses exposed student API data via unauthenticated endpoints.
2. For each student record, the MD5 hash of their password is visible.
3. The attacker uses publicly available MD5 lookup tools or precomputed rainbow tables to reverse the hash into the original password.
4. The cracked password may allow login to other Ednovation services or reused accounts.

Example MD5 hash:
```json
{
  "username": "student001",
  "password": "e10adc3949ba59abbe56e057f20f883e"
}
```
This is the MD5 of `123456`

## Impact

- Passwords for children or parents may be trivially cracked.
- If passwords are reused across other platforms (email, messaging apps), this could lead to wider compromise.
- Violates OWASP recommendations and industry standards for password storage.

## **Security Best Practices Recommended**:

- Immediately remove MD5 hashing and switch to secure password hashing schemes:
    - bcrypt, scrypt, or argon2
- Never expose password hashes in API responses.
- Enforce stronger password complexity requirements.
- Conduct a full audit of current password management infrastructure.

## Timeline

- **2024-07**: Discovered while analyzing leaked student records via unauthenticated APIs, responsibly reported to vendor
- **2024-08**: Vendor acknowledged and deployed a patch
- **2025-07**: Public disclosure and CVE request initiated


