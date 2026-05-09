---
title: "ADG Unblock Summary"
date: 2026-05-09
draft: false
categories: ["dns-networking"]
tags: []
---

# Chat Summary: Domain Unblock Rules

## Context

A list of domains shown in a blocklist/allowlist UI (likely a DNS filter or ad blocker) needed to be converted into unblock rules using `@@||domain^` syntax.

## Unblock Rules

Add the following to your filter list to unblock all domains:

```
@@||www.powershellgallery.com^
@@||teams.microsoft.com^
@@||appx.transient.amazon.co.uk^
@@||fls-eu.amazon.com^
@@||unagi-na.amazon.com^
@@||session.mshopbugsnag.irm.amazon.dev^
@@||get.activated.win^
@@||c.microsoft.com^
@@||vlscppe.microsoft.com^
@@||az416426.vo.msecnd.net^
@@||s.youtube.com^
```

## Syntax Notes

- `@@||domain^` unblocks a domain and all its subdomains.
- The `*` prefix shown in the UI is covered by the `||` syntax — no extra wildcard needed.
