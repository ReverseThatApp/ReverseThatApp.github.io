---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Unauthenticated Internal API Testing Interface Exposing Hardcoded Production Credentials"
date: 2025-07-29
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, authentication, authentication-bypass, credential-exposure, ednovation, vulnerability, aimath, api]
permalink: /disclosures/unauthenticated-api-testing-interface-hardcoded-production-credentials/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczOVCW-SSaqJpPxtV8q_KFwdRo4-AakK2YBKpOQwhbFhmklJMjVxdhZ9ezjNtEP72qCcIjL2D7I23R7kHNJjz4WVWntz1P3vW-akAjkQUqHUN10wFkGGXisatTAx25DvmyXHafLFPXKe4l1ZphdRo3T5=w1578-h1108-s-no-gm
    alt: Test API Form
---

## Summary

A **publicly accessible internal API testing tool** was discovered on Ednovation's development subdomain. This tool contains **hardcoded production credentials** and allows unauthenticated users to directly interact with sensitive backend APIs.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

While reviewing the directory listing at `https://ednoapi-dev.ednoland.com/vrcreate/` one file stood out: `test_API_form.php`. Clicking on it revealed a fully functional **API testing web form**.

This interface is intended for internal developers to test API calls but is **exposed to the public internet without any authentication**.

## Affected Product

- **Vendor**: [Ednovation](https://ednovation.com)
- **Subdomain**: `https://ednoapi-dev.ednoland.com/vrcreate/test_API_form.php`
- **Component**: Internal API testing tool
- **Environment**: Development environment accessible publicly

## Vulnerability Details

- The API testing form allows:
  - Direct API calls to internal backend endpoints.
  - Selection and execution of various API actions.
- **Hardcoded credentials** are embedded in the form fields:
  - Valid student/teacher usernames and passwords.
  - These credentials work **against the production environment**.
- No authentication is required to access the page.

![Test API Form](https://lh3.googleusercontent.com/pw/AP1GczOVCW-SSaqJpPxtV8q_KFwdRo4-AakK2YBKpOQwhbFhmklJMjVxdhZ9ezjNtEP72qCcIjL2D7I23R7kHNJjz4WVWntz1P3vW-akAjkQUqHUN10wFkGGXisatTAx25DvmyXHafLFPXKe4l1ZphdRo3T5=w1578-h1108-s-no-gm)
_**Figure: Test API Form**_

## Exploitation

1. Attacker visits `https://ednoapi-dev.ednoland.com/vrcreate/test_API_form.php`
2. Reads hardcoded login credentials from pre-filled form fields.
3. Uses these credentials to:
- Log into production API endpoints.
- Access or modify sensitive student/teacher data.

## Impact

- **Full compromise** of student/teacher accounts in production.
- **Exposure of sensitive PII** in live systems.
- **Potential compliance violations** under PDPA, GDPR, and similar laws.

## Mitigation

- Immediately restrict access to internal testing tools.
- Remove hardcoded credentials from all environments.
- Use separate non-production accounts for testing.
- Secure development subdomains with authentication or IP allowlisting.

## Timeline

- **2024-08**: Vulnerability discovered during related infrastructure testing, responsibly reported to vendor
- **2024-09**: Vendor acknowledged and deployed a patch  
- **2025-07**: Public disclosure and CVE request initiated