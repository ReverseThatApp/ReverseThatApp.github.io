---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] ParentCommApp Insecure Direct Object Reference (IDOR)"
date: 2025-07-26
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, idor, pii-exposure, ednovation, parentcommapp, vulnerability, ios, api]
permalink: /disclosures/parentcommapp-insecure-direct-object-reference-idor/
---

## Summary

A critical **Insecure Direct Object Reference (IDOR)** vulnerability was discovered in Ednovation’s *ParentCommApp* for iOS, allowing authenticated users to retrieve private data of other children by modifying a predictable `childId` parameter in an API request.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

---

## Background & Discovery

This issue was discovered while testing the iOS version of *ParentCommApp* after the vendor had deployed fixes for a previously reported broken authentication implementation. Although the latest app version uses server-generated JWT tokens for request authorization, the backend still relies on unvalidated `childId` values passed via URL query parameters.

The API continues to trust these user-supplied identifiers without verifying whether the requesting party is authorized to access the corresponding resource, leading to a classic IDOR vulnerability.

## Affected Product

- **App**: ParentCommApp (iOS)
- **Vendor**: [Ednovation](https://ednovation.com)
- **Platform**: Backend API — Child Info Endpoint
- **Version**: Affected prior to 2022 remediation

## Vulnerability Details

The API endpoint used to fetch student details takes an input parameter `childId`, which is a predictable integer (e.g., `childId=101`, `102`, etc). While the app includes a valid JWT token in its request headers, the server does not validate whether the requester has permission to access the `childId` specified.

Any authenticated user can increment the value of `childId` to access other children’s personal records.

## Exploitation

An attacker with a valid login can craft a request like:

```http
POST /matp/parentApp/parentAppRequest.php
Authorization: Bearer <valid_token>

action=getChildInfo&childId=1234
```

Then change the `childId` to:

```http
POST /matp/parentApp/parentAppRequest.php
Authorization: Bearer <valid_token>

action=getChildInfo&childId=1235
```

The server will return private data for the new child without any authorization check. This can be repeated to enumerate other children’s information.

## Impact

- Unauthorized access to sensitive personal information of children
- Privacy violations against students and families
- Regulatory risk (e.g., PDPA, GDPR violations)
- Enables enumeration attacks across the entire user base

## Mitigation

- Implement proper object-level access control in the backend
- Validate that the `childId` belongs to the authenticated user
- Avoid reliance on predictable numeric identifiers in APIs

## Timeline

- **2022-09**: Issue discovered and responsibly reported to vendor  
- **2022-09**: Vendor acknowledged and deployed a patch  
- **2025-07**: Public disclosure and CVE request initiated  