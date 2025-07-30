---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Broken JWT Authentication – Hardcoded Shared Secret in ParentCommApp (iOS)"
date: 2025-07-26
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, jwt, broken-auth, ednovation, parentcommapp, vulnerability, ios, api]
permalink: /disclosures/parentcommapp-broken-jwt-hardcoded-shared-secret/
---

## Summary

A **critical design flaw** was discovered in the JWT authentication implementation used by **ParentCommApp**, an iOS app developed by **Ednovation**. The app includes a **hardcoded shared secret key** used for **client-side JWT token signing**, allowing any attacker to generate valid tokens and impersonate legitimate users.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

This issue was discovered while testing the app for potential vulnerabilities and analyzing the app’s network traffic post-login using Burp Suite. The inspection revealed that the app employed **JWT tokens** for API authentication. However, it was found that the tokens were being generated **on the client side** using a secret embedded in the app bundle — a fundamental misuse of JWT authentication.

## Affected Product

- **Vendor**: [Ednovation](https://ednovation.com)
- **App**: ParentCommApp (iOS)
- **Platform**: iOS (client-side implementation flaw)
- **Version**: Affected prior to 2022 remediation

## Vulnerability Details

The updated app switched to JWT-based request validation. However, the `secretKey` used to sign the JWTs was hardcoded in the app bundle (`Application.xml`):

```xml
<secretKey>REDACTED_XXX</secretKey>
```

This key is shared between the app and backend to sign and validate JWT tokens. This is a misuse of the JWT model:
- JWT signing must be performed by the server
- Clients should only receive JWTs after authenticated login
- A hardcoded shared secret makes token spoofing trivial

This setup enables an attacker to forge valid JWTs and access any API endpoints intended for authenticated users.

## Exploitation

1. Extract .ipa file on the iOS device (can use jailbroken device)
2. Locate and read `Application.xml` to retrieve the <secretKey>
3. Use the key to forge a valid JWT using standard libraries
4. Submit the token to backend endpoints to impersonate any user

## Impact

- Complete authentication bypass
- Unauthorized access to sensitive child, parent, and teacher data
- Loss of data confidentiality and privacy

## Mitigation

- Never store JWT secrets on the client side
- Server should handle all token generation and validation
- Use asymmetric signing (e.g., RS256) with a private signing key on server
- Treat the client as untrusted at all times

## Timeline

- **2022-09**: Issue discovered and responsibly reported to vendor  
- **2022-09**: Vendor acknowledged and deployed a patch  
- **2025-07**: Public disclosure and CVE request initiated  