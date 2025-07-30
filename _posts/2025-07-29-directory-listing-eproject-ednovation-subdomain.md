---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Directory Listing Exposure on Ednovation's Production Subdomain EProject"
date: 2025-07-29
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, directory-listing, misconfiguration, ednovation, vulnerability]
permalink: /disclosures/directory-listing-exposure-subdomain-eproject/
---

## Summary

A **directory listing exposure** was identified on Ednovation's **production** subdomain `eproject.ednoland.com`.  
The vulnerability reveals sensitive server directories, configuration folders, and internal API structures to any unauthenticated user.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

While conducting security testing on Ednovation's ecosystem, the production subdomain `https://eproject.ednoland.com` was scanned using a standard directory enumeration tool on `kali`:

```bash
gobuster dir -u https://eproject.ednoland.com \
-w /usr/share/wordlists/dirbuster/directory-list-1.0.txt -t 4
```

The scan revealed multiple publicly browsable directories.

## Affected Product
- **Vendor**: [Ednovation](https://ednovation.com)
- **Subdomain**: `https://eproject.ednoland.com`
- **Component**: Web server configuration (production environment)
- **Environment**: Live production system accessible on the public internet

## Vulnerability Details

The following directories are exposed via public directory listing:
```
/.config
/.cache
/html
/api
/vendor
/toolkits
/API
/tmp
```

## Risk Analysis
- **/.config** — contains sensitive configuration files, log files
- **/vendor** — Expose library and framework versions for targeted attacks.
- **/tmp** — contains a few `.php` files that can be trigger to return content
- **/toolkits** — contains `conf.php` file

Exposing the directory structure on a production system significantly increases the risk of targeted exploitation.

## Exploitation

An attacker could:
1. Enumerate files in these directories to identify exploitable scripts.
2. Locate configuration or cached files that may leak secrets.
3. Map backend APIs for targeted endpoint attacks.
4. Use temporary or upload directories to plant or retrieve malicious files.

## Impact

- Increased attack surface for production infrastructure.
- Potential exposure of sensitive configuration, API details, and operational files.
- Facilitation of further targeted exploitation.

## **Security Best Practices Recommended**:

- Disable directory listing in the web server configuration (Options -Indexes for Apache).
- Restrict public access to non-essential directories.
- Perform a security review of exposed directories for sensitive data.

## Timeline

- **2024-08**: Vulnerability discovered during related infrastructure testing and responsibly reported to vendor
- **2024-09**: Vendor acknowledged and deployed a patch  
- **2025-07**: Public disclosure and CVE request initiated


