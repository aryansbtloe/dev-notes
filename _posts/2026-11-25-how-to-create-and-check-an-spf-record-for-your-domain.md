---
layout: post
title: "How to Create and Check an SPF Record for Your Domain"
date: 2026-11-25 10:00:00 +0530
author: Alok Singh
description: "Build an SPF TXT record for Google Workspace, Microsoft 365 and other senders, check the record a domain publishes today, and stay under the 10-lookup limit."
canonical_url: "https://bestlatestapps.com/blog/how-to-create-and-check-an-spf-record-for-your-domain/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-create-and-check-an-spf-record-for-your-domain/).

{% raw %}

Since Gmail and Yahoo tightened their sender rules in 2024, mail from domains without proper SPF, DKIM and DMARC lands in spam far more often. SPF is the first piece: a DNS TXT record listing which servers may send mail for your domain. [SPF record generator](https://bestlatestapps.com/tools/spf-record-generator/) builds one from checkboxes and also checks what a domain currently publishes.

## Steps

1. Enter your domain and tick the services that send mail for it (your mailbox provider, newsletter tool, CRM, helpdesk).
2. Add extra senders under **More senders**: IP addresses, your domain's own A or MX servers, or a redirect.
3. Read the generated TXT record.
4. Press **Check the published record** to see what the domain publishes today.

## Anatomy of a record

Every SPF record starts with `v=spf1`. Each ticked service adds an `include:` for that service's SPF host. You can add `ip4:` and `ip6:` addresses, and `a` or `mx` to authorise your domain's own servers.

The record ends with an `all` mechanism:

- `-all`: fail anything not listed (strict; recommended once you're confident).
- `~all`: soft fail, so unlisted servers are suspicious but not rejected (a common starting point).
- `?all`: neutral, which says nothing useful.
- `+all`: pass everything. Never use this; it lets anyone send as you.

A `redirect=` is only written when the `all` mechanism is left empty, because a record that already has `all` ignores redirect.

## The 10-lookup limit

This is the trap that breaks SPF for growing companies. Every `include`, `a`, `mx`, `ptr`, `exists` and `redirect` costs a DNS lookup, and includes can contain more includes. Receivers stop at **10 lookups** and return a permanent error, which effectively means SPF fails. The checker counts lookups as it follows includes (the domain's own TXT query is free), flags loops, and spots a second SPF record. Having two SPF records is itself an error; merge them into one.

If you're over 10, remove services you no longer use, replace `a`/`mx` with explicit IPs where practical, or ask a vendor whether they offer a narrower include.

## Example: a small business

You use Google Workspace for mail, a newsletter tool, and an invoicing app that emails customers. Tick those three, choose `~all`, publish the TXT record at your DNS host, then press Check the published record to confirm it's live and under 10 lookups. After a couple of weeks of clean DMARC reports, tighten to `-all`.

## What SPF doesn't do

SPF checks the server that sent the message against the envelope sender domain, which users never see. It doesn't protect the visible From address on its own. That's DMARC's job, which relies on SPF or DKIM aligning with the From domain. Set up DKIM with each service too, then add DMARC.

## Rolling out changes safely

DNS changes can take time to spread, and a typo in SPF can send legitimate mail to spam. Lower the TTL on the record a day before you edit it, make the change, then send test messages to a Gmail and an Outlook address and check the headers for `spf=pass`. Keep the old record value in a note so you can roll back quickly if something you forgot about (a billing system, a website contact form) starts failing.

## Network use

Building the record happens in the tab. **Checking** a domain sends that domain name to Cloudflare's public DNS resolver to read its current TXT records, which is how any DNS lookup works. Nothing else is sent. Google ads appear on the page, as on the rest of the site.

## FAQ

**What does an SPF record do?**
Lists the servers allowed to send mail for your domain.

**What does `-all` mean?**
Fail every server not listed. `~all` is a soft fail.

**How are lookups counted?**
include, a, mx, ptr, exists and redirect each count, up to 10.

**What if my domain already has SPF?**
The check shows it, follows includes, and flags loops or duplicates.

Fix deliverability at the root: build your record with **[SPF record generator](https://bestlatestapps.com/tools/spf-record-generator/)**.

*— Alok Kumar Singh*

{% endraw %}
