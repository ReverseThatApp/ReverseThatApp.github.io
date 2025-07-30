---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Unauthenticated API Exposure in Ednovation ParentCommApp"
date: 2025-07-25
cve: "Reported"
status: "Fixing"
categories: [Disclosure]
tags: [cve-requested, CVE, unauthenticated-api, pii-exposure, ednovation, parentcommapp, vulnerability, ios, api, authentication]
permalink: /disclosures/parentcommapp-api-auth-bypass/
---

## Summary

A critical security vulnerability was discovered in **ParentCommApp** — an iOS mobile application used by preschools and parents for communication within the **Ednovation** ecosystem.

The vulnerability allows **unauthenticated access to sensitive backend API endpoints**, exposing personal information of students, parents, and teachers. The issue was rooted in insecure server-side logic and could be exploited without any login or authentication token.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

This issue was discovered while testing the ParentCommApp on iOS, where traffic was intercepted using **Burp Suite**. After a successful login, it was observed that subsequent requests to sensitive API endpoints lacked any authentication token in the HTTP headers.

Further manual testing confirmed that these endpoints were accessible directly by unauthenticated clients — indicating a server-side design flaw rather than a client implementation issue.

## Affected Product

- **Vendor**: [Ednovation Pte Ltd](https://ednovation.com)
- **Affected Component**: Backend API server supporting ParentCommApp
- **Mobile App (Client)**: ParentCommApp (iOS)
- **Version**: Affected versions prior to mid-2022 patch deployment
- **Platform**: iOS (used to trigger the vulnerability)

## Vulnerability Details

One backend API endpoint exposed in the app's network traffic allows fetching sensitive student and parent information **without requiring any form of authentication**. This endpoint accepts a `childId` parameter and returns full records even when called by anonymous users.

### Sensitive Data Leaked

- Student full name and identifiers  
- Parent full name, **phone number**, and **email address**  
- **Parent plaintext password**  
- Class, teacher, and staff information (including **plaintext passwords**)  
- Other personally identifiable information (PII) of children and families

## Exploitation

- Attackers require only a known or guessed `childId` to query the API.
- No authentication headers (e.g., session token, JWT) are required.
- The API responds with sensitive records regardless of user identity or session.

This represents a **critical broken authentication flaw** with serious implications for confidentiality, privacy, and legal compliance.

## Impact

- 📛 **Massive confidentiality breach** affecting children, parents, and staff  
- 📈 **High risk of data enumeration** due to use of sequential or predictable IDs  
- ⚖️ **Possible violations of data protection regulations** (e.g., **PDPA**, **GDPR**)  
- 🧸 Exposure of **child-related sensitive data**, raising serious ethical concerns

## Mitigation

The vendor has since migrated to new API endpoints which enforce proper authentication. The legacy endpoints that previously served this unauthenticated data have now been deprecated and disabled.

Mitigation

- Enforce token-based authentication on *all* sensitive APIs  
- Validate access control contextually (e.g., user roles, ownership of child data)  
- Do not expose plaintext passwords or sensitive tokens under any circumstance

## Timeline

- **2022-06**: Initial discovery and report sent to vendor  
- **2022-07**: Issue acknowledged and patched in new backend endpoint design  
- **2025-07**: Legacy vulnerable endpoints still observed active; re-reported and resolved  
- **2025-07**: Public disclosure and CVE request submitted
