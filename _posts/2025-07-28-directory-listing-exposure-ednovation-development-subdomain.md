---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] Directory Listing Exposure on Ednovation's Development Subdomain"
date: 2025-07-28
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security,directory-listing, misconfiguration, ednovation, vulnerability]
permalink: /disclosures/directory-listing-exposure-ednovation-development-subdomain/
---

## Summary

A publicly accessible **directory listing** was found on Ednovation's development subdomain, exposing the internal file structure of a web application, including PHP request handlers, code libraries, and uploaded assets. This misconfiguration can assist attackers in identifying exploitable files and endpoints.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

While testing Ednovation’s iOS ParentCommApp backend, additional reconnaissance of related subdomains revealed that the development subdomain `https://ednoapi-dev.ednoland.com/vrcreate/` was configured with **Apache directory listing enabled**. This allows anyone to browse directories and access file and folder names without authentication.

## Affected Product

- **Vendor**: [Ednovation](https://ednovation.com)
- **Subdomain**: `https://ednoapi-dev.ednoland.com/vrcreate/`
- **Component**: Web server configuration
- **Environment**: Development environment accessible on the public internet

## Vulnerability Details

Directory listing revealed:
- Multiple `.php` request handler files such as:
  - `Upload.php`
  - `UserEnrichment.php`
  - `test.php`  
- Sensitive directories:
  - `commonlib/` (shared code library)
  - `themes/` (site styling assets)
  - `uploads/` (user-uploaded content)

Although clicking on `.php` files did not reveal their source code, knowledge of their existence provides attackers with valuable reconnaissance for crafting targeted attacks.

## Exploitation

An attacker could:
1. Browse the directory structure to identify likely API endpoints or admin tools.
2. Target exposed `.php` handlers with parameter tampering or injection attacks.
3. Access uploaded files directly from `/uploads/` if paths are predictable.

## Impact

- **Increased attack surface awareness** for malicious actors.
- **Potential exposure** of sensitive uploaded files.
- **Facilitation of targeted exploitation** of backend scripts.

## **Security Best Practices Recommended**:

- Disable directory listing at the web server level (`Options -Indexes` in Apache).
- Restrict access to development environments with authentication.
- Move non-public assets and scripts to directories inaccessible via the public internet.

## Timeline

- **2022-08**: Vulnerability discovered during related infrastructure testing and responsibly reported to vendor
- **2022-08**: Vendor acknowledged and deployed a patch  
- **2025-07**: Disclosure and CVE request initiated
