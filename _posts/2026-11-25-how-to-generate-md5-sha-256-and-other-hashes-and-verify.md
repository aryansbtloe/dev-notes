---
layout: post
title: "How to Generate MD5, SHA-256 and Other Hashes, and Verify a Download"
date: 2026-11-25 18:00:00 +0530
author: Alok Singh
description: "Compute MD5, SHA-1, SHA-256, SHA-384 and SHA-512 hashes of text or files in your browser, verify a download's checksum, and generate HMACs, with the newline gotcha explained."
canonical_url: "https://bestlatestapps.com/blog/how-to-generate-md5-sha-256-and-other-hashes-and-verify/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-generate-md5-sha-256-and-other-hashes-and-verify/).

{% raw %}

You download an ISO, a firmware update or a CLI binary, and the project page lists a SHA-256 checksum. Most people skip checking it because opening a terminal and remembering `shasum -a 256` versus `certutil -hashfile` is a hassle. [Hash generator](https://bestlatestapps.com/tools/hash-generator/) hashes text or files in the browser, compares against a checksum you paste, and does HMAC too.

## What it supports

- **Algorithms:** MD5, SHA-1, SHA-256, SHA-384, SHA-512.
- **Modes:** plain hash, or HMAC with a key.
- **Inputs:** text (UTF-8 by default, or hex/Base64 when you already have raw bytes) or a file.
- **Verification:** paste an expected checksum and see whether it matches.

## Steps

1. Type text or choose a file.
2. Tick the algorithms you want, plus HMAC if you have a key.
3. Copy the hash, or paste the expected checksum to compare.

## Verifying a download

Choose the file, tick SHA-256 (or whatever the publisher lists), and paste their checksum. The comparison works out which algorithm the pasted value belongs to from its length (64 hex characters means SHA-256, for instance) and tells you whether it matches.

Files are read in chunks inside the tab with a progress indicator, so even multi-gigabyte images work without loading everything into memory at once. Stop ends the read if you picked the wrong file.

A match means the file you have is byte-for-byte what the publisher hashed. It doesn't prove the publisher is trustworthy, so get the checksum from the official site over HTTPS, not from a mirror.

## The trailing newline gotcha

This trips up nearly everyone at some point. In a terminal, `echo hello | sha256sum` hashes "hello" **plus a newline**, because `echo` adds one. That gives a different hash from the five characters "hello".

The tool keeps exactly what you typed by default. There's an option to add a trailing newline, but it's off until you tick it. For reference, the MD5 of plain `hello` is `5d41402abc4b2a76b9719d911017c592`. If your terminal gives something else, the newline is why (use `echo -n` or `printf`).

Other reasons a checksum doesn't match: one changed byte anywhere in the input, or comparing the hash of hex text against the hash of the bytes it represents.

## HMAC

HMAC combines a secret key with the message, which is how webhook signatures from Stripe, GitHub, Razorpay and others work. When a webhook signature check fails in your code, paste the raw body and your secret here, choose SHA-256, and compare with the header value. If this matches and your code doesn't, your code is hashing a re-serialised or trimmed body instead of the raw bytes.

The HMAC key stays in the form, isn't added to the page link and isn't saved with your inputs.

## Which algorithm to use

- **SHA-256:** the sensible default for checksums and HMAC.
- **SHA-384 / SHA-512:** fine, longer output; some standards require them.
- **MD5 and SHA-1:** still common for legacy checksums and non-security deduplication, but broken for security purposes. Don't use them for signatures or anything an attacker might target.

None of these are suitable for storing passwords on their own; use bcrypt, scrypt or Argon2 for that.

## Privacy

According to the page, everything runs in the tab: pasted text stays there, files are read in pieces locally, and the HMAC key is kept out of the URL. Google ads appear on the page as on the rest of the site.

## FAQ

**Which algorithms are available?**
MD5, SHA-1, SHA-256, SHA-384 and SHA-512, plain or as HMAC.

**Why doesn't my checksum match?**
A changed byte, a trailing newline, or hex-versus-text confusion.

**Can it hash big files?**
Yes, they're read in chunks with progress.

**Where does my HMAC key go?**
Nowhere; it stays in the form.

Check the next download before you run it: **[Hash generator](https://bestlatestapps.com/tools/hash-generator/)**.

*— Alok Kumar Singh*

{% endraw %}
