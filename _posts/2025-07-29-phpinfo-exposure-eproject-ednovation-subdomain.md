---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] phpinfo() Exposure on Ednovation's Production Subdomain"
date: 2025-07-29
cve: "Reported"
status: "Fixed"
categories: [Disclosure]
tags: [cve-requested, CVE, web-security, phpinfo, misconfiguration, ednovation, vulnerability]
permalink: /disclosures/phpinfo-exposure-eproject-ednovation-subdomain/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPjMvmjeSF8jSEM4iLr5K-aajfxJrZzLeVKWy8U97c_gwJ5T9dCnFZtLuT0iVrMsIoW3KrmT9YzpL8_AQFctkSHHUYJy2Wvuk95yyfE_WijJiFmipwihlILQ5zD3XdK8z9Ck3NJ79vRzhQx3ujxGLkH=w2126-h1544-s-no-gm
    alt: phpinfo()
---

## Summary

A publicly accessible PHP configuration page (`phpinfo()`) was discovered on Ednovation's **production** subdomain `eproject.ednoland.com`.  
The exposure reveals **sensitive server configuration details**, including environment variables, PHP extensions, loaded modules, and potentially database connection details.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

During a directory listing review on the production subdomain `https://eproject.ednoland.com/toolkits/` one of the files listed was `conf.php`.  

Accessing `https://eproject.ednoland.com/toolkits/conf.php` resulted in a full **`phpinfo()`** output dump.

## Affected Product

- **Vendor**: [Ednovation](https://ednovation.com)
- **Subdomain**: `https://eproject.ednoland.com`
- **Component**: PHP configuration exposure
- **Impact scope**: Live production system

## Vulnerability Details

The exposed **`phpinfo()`** page contains:

- Full PHP version and build details
- Loaded extensions and their versions
- PHP configuration directives
- Environment variables
- Include paths
- HTTP headers from the request
- Server OS details
- Potential connection credentials (if stored in environment variables)

![phpinfo() exposure](https://lh3.googleusercontent.com/pw/AP1GczPjMvmjeSF8jSEM4iLr5K-aajfxJrZzLeVKWy8U97c_gwJ5T9dCnFZtLuT0iVrMsIoW3KrmT9YzpL8_AQFctkSHHUYJy2Wvuk95yyfE_WijJiFmipwihlILQ5zD3XdK8z9Ck3NJ79vRzhQx3ujxGLkH=w2126-h1544-s-no-gm)
_**Figure: phpinfo() exposure**_

## Exploitation

An attacker can leverage this exposure to:

1. **Fingerprint** the PHP version, extensions, and server OS for targeted attacks.
2. **Locate sensitive values** in environment variables (e.g., database passwords, API keys).
3. **Gain insights** into server paths, configuration weaknesses, and enabled modules.

## Impact

- **Sensitive information disclosure** — increases risk of targeted exploitation.
- **Facilitates further attacks** such as PHP exploits, library-specific vulnerabilities, or privilege escalation.
- **Production system exposure** — significantly raises the attack surface.

## Mitigation

- Immediately remove public access to `conf.php`.
- Avoid deploying diagnostic or debug pages to production environments.
- Restrict sensitive files behind authentication and server configuration rules.

## Timeline

- **2025-07**: Vulnerability discovered after reviewing public directory listing and responsibly reported to vendor
- **2025-07**: Vendor acknowledged and deployed a patch
- **2025-07**: Public disclosure and CVE request submitted.