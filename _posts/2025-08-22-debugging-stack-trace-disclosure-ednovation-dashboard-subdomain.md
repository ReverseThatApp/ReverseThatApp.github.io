---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Debugging Stack Trace Disclosure on Ednovation Dashboard Subdomain"
date: 2025-08-22
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, laravel, debugging, information-disclosure, ednovation, vulnerability]
permalink: /disclosures/debugging-stack-trace-disclosure-ednovation-dashboard-subdomain/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPK13CCBzrAV_TNPQ9MMRlrj-GoGPBrVhbz0THTnMe4wUaArcUL3Nk9PHTkyuwfL99tle881xQtQ1L40rJTG6Ir3Dh1lo_UuPZlLomFtsdiUVOiMli7EX3_bDi0n330T9o4DEKXtdsWoKg_q9YtPGlL=w2398-h1354-s-no-gm
    alt: Debugging Stacktrace
---

## Summary

A **debugging error disclosure vulnerability** exists on Ednovation's `https://dashboard.ednoland.com/` subdomain.  
By requesting a non-existent path, the application returns a full **Laravel exception stack trace**, exposing sensitive framework and server information.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

While testing the **ParentCommApp** iOS mobile application, a related Ednovation subdomain was identified `https://dashboard.ednoland.com/`

Accessing an invalid route such as `https://dashboard.ednoland.com/test` or when accessing a file download endpoint (e.g., `/public/assets/[type]/[file]` triggers a **detailed Laravel error page**, revealing internal application details.


## Affected Product

- **Vendor**: [Ednovation](https://ednovation.com)
- **Subdomain**: `https://dashboard.ednoland.com`
- **Technology**: Laravel PHP Framework
- **Environment**: Production

## Vulnerability Details

The application displays a full Laravel exception trace:

- Internal PHP file paths  
- Class, method and parameter names  
- Laravel routing internals  
- Middleware chain  
- Application kernel details  
- Potentially framework version information

![Debuging Stacktrace](https://lh3.googleusercontent.com/pw/AP1GczMV892blejLCojTzZw13_BKlneWy9PhW1CQoA1quOyxu54NJCcttoR4GQDWNWsFmleY_JQFrI8UGz1V7eJZvIOFOtPM7khipc0JQ5y9Kh7YoWfh6H6lR1Zyea_vxzIcoSXYSJzQpcsZM36FQ_RhuaiR=w2126-h780-s-no-gm)
_**Figure: Debuging Stacktrace**_

![Download File Stacktrace](https://lh3.googleusercontent.com/pw/AP1GczPK13CCBzrAV_TNPQ9MMRlrj-GoGPBrVhbz0THTnMe4wUaArcUL3Nk9PHTkyuwfL99tle881xQtQ1L40rJTG6Ir3Dh1lo_UuPZlLomFtsdiUVOiMli7EX3_bDi0n330T9o4DEKXtdsWoKg_q9YtPGlL=w2398-h1354-s-no-gm)
_**Figure: Download File**_

## Exploitation

Attackers can use this vulnerability to:

- Identify the PHP/Laravel version and framework internals.  
- Map internal class structures, middleware, and routing logic.  
- Fingerprint the operating system and server configuration.
- Craft malicious inputs for path traversal attacks, potentially accessing sensitive files like configuration files or source code.  
- Develop targeted exploitation strategies.

## Impact

- **Information disclosure**: Reveals sensitive environment details.  
- **Increased attack surface**: Assists in developing targeted attacks against known vulnerabilities in the Laravel/PHP stack.  
- **Potential pivot point**: May aid in chaining with other vulnerabilities.

## Mitigation

- Disable detailed error pages in production by setting `APP_DEBUG=false` in Laravel `.env`.  
- Ensure a generic 404/500 error handler is configured.  
- Restrict verbose stack traces to local development environments only.

## Timeline

- **2025-07**: Vulnerability discovered during mobile app-related subdomain testing and responsibly reported to vendor
- **2025-08**: Vendor acknowledged and deployed a patch  
- **2025-08**: Public disclosure and CVE request initiated
