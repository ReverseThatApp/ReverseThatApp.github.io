---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Public Data Exposure via Broken Auth in Aimath Web App"
date: 2025-07-27
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, authentication, idor, pii-exposure, ednovation, vulnerability, ios, aimath, api]
permalink: /disclosures/aimath-web-broken-auth/
---

## Summary

A critical security flaw was identified in the **Aimath Web App**, a math learning platform operated by Ednovation, which exposes **student and parent personal data via unauthenticated GET APIs**. The flaw exists due to improperly protected endpoints discovered in the client JavaScript bundle and leads to widespread data leakage.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

While analyzing the JavaScript bundle (`bundle.js`) of the Aimath Web App hosted on a separate subdomain from ParentCommApp, several internal API endpoints were discovered. These endpoints were called by a shared `processGet()` function and were entirely **unauthenticated**, accepting direct requests from any unauthenticated source.

This issue was uncovered during passive testing of Ednovation's broader ecosystem of educational platforms and stems from unprotected API usage in a production-facing web application.

## Affected Product

- **App**: Aimath Web App
- **Vendor**:  [Ednovation](https://ednovation.com)
- **Subdomain**: `https://aimath.ednoland.com/`
- **Component**: Client-side JavaScript (`bundle.js`) and associated API backend
- **Version**: Unknown (latest production build prior to discovery)

## Vulnerability Details

The vulnerability stems from usage of a common JavaScript function, `processGet()`, that performs HTTP GET requests to backend endpoints without requiring any authentication token, session cookie, or user credential.

Endpoints include:

- Fetching all classes
- Fetching all students in a class
- Fetching student details (including personal data)

### Data Leaked Includes:

- Student ID, username, display name
- Parent full name, mobile number, email address
- Student profile photo URL
- **Encrypted password** and even **plaintext password**
- Class group and organization details

## Exploitation

- The attacker can analyze the JavaScript code served from `aimath.ednoland.com` to extract internal API paths.
- Directly calling these endpoints via GET requests (e.g., using curl or a browser) returns **raw JSON data**.
- No authentication is required, and the server does not verify session or JWT.

Example:

```http
GET https://aimath.ednoland.com/classes/students/2
```

Returns full student data for class `2`, including parent contact information.

## Impact

- Privacy violation: Exposure of student and guardian PII
- Sensitive credential exposure: Passwords appear in both encrypted and plaintext forms
- Data enumeration risk: Entire student directory can be scraped unauthenticated
- Violation of data protection obligations (e.g., PDPA, GDPR)

## **Security Best Practices Recommended**:

- All APIs should enforce access control using secure authentication mechanisms (e.g., session tokens, JWT).
- Remove plaintext password storage from both frontend and backend systems.
- Obfuscate or minimize exposed JavaScript where feasible to avoid exposing internal endpoints.
- Conduct a full audit of client-side exposed API calls.

## Timeline

- **2022–06**: [Related flaws discovered in other Ednovation products (ParentCommApp iOS)](/disclosures/parentcommapp-api-auth-bypass/)
- **2022-07**: This distinct issue discovered in Aimath Web App
- **2022-08**: Vendor notified; acknowledged issue
- **2025-07**: Disclosure and CVE request initiated  