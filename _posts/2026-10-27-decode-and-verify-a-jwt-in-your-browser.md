---
layout: post
title: "How to Decode and Verify a JWT in Your Browser (Without Pasting It Into a Server)"
date: 2026-10-27 10:00:00 +0530
author: Alok Singh
description: "Read a JSON Web Token's header and payload, see exp, iat and nbf as real dates, check whether it has expired, and verify its signature with Web Crypto, all inside a browser tab."
canonical_url: "https://bestlatestapps.com/blog/decode-and-verify-a-jwt-in-your-browser/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/decode-and-verify-a-jwt-in-your-browser/).

{% raw %}
Sooner or later every app developer stares at a 401 and a token that looks like `eyJhbGciOi...`. Is it expired? Is the audience wrong? Was it signed with the key you think it was? You could decode it by hand with Base64, but it's easy to get the URL-safe alphabet wrong, and `exp: 1793462400` doesn't tell you much at a glance.

[JWT decoder](https://bestlatestapps.com/tools/jwt-decoder/) on Best Latest Apps answers those questions in a browser tab.

## What a JWT is, briefly

A JSON Web Token has three Base64url-encoded parts separated by dots: a **header** (which algorithm signed it), a **payload** (the claims, such as who it's for and when it expires) and a **signature**. Anyone can decode the first two; only the right key can produce or check the signature.

## What the tool does

- Shows the **header** and **payload** as formatted JSON.
- Converts the timestamps, **exp** (expiry), **iat** (when it was issued) and **nbf** (not valid before), from Unix seconds into your local time, and says how far in the past or future each one is.
- Explains the standard claims you'll meet most, like **iss** (issuer), **sub** (subject) and **aud** (audience).
- Tells you whether the token has **expired**, isn't valid yet, or is within its validity window.
- **Checks the signature** using the browser's built-in Web Crypto: give it the shared secret for the HMAC algorithms (HS256/384/512), or a PEM public key for RSA and ECDSA ones (RS256, PS256, ES256 and the bigger key sizes). There's a checkbox for secrets that are Base64-encoded.

## When you'd use it

- Debugging a 401 or 403 from an API
- Checking what claims your identity provider actually issues
- Confirming an APNs provider token's header and timestamps
- Checking that a token was signed by the key you expect after a key rotation

## How to decode a JWT, step by step

1. Open [bestlatestapps.com/tools/jwt-decoder/](https://bestlatestapps.com/tools/jwt-decoder/).
2. Paste the token into **JSON Web Token**.
3. Look through the decoded header and payload, plus the table of claims, with the expiry status next to them.
4. To check the signature, paste the secret (HS algorithms) or the public key PEM (RS, PS, ES algorithms) under **Verify the signature**. Tick **The secret is Base64-encoded** if it is.
5. Press **Verify**.

## Privacy and how it works

Pasting a live credential into a random website is a bad habit, so here's what the page says and what I checked. According to the page, anything you paste (tokens and keys alike) stays in the tab and never goes into the URL or browser storage. The script behind it has no code that sends data over the network: decoding is plain Base64url and JSON parsing, and verification uses the browser's built-in `crypto.subtle`. The page also shows Google ads, which can use cookies, separately from the token.

Even so, the page's own advice is sensible: treat live tokens as passwords and use test tokens when you can.

## Decoding is not verifying

This is the most common JWT mistake, so it's worth saying plainly. **Decoding only reads a token.** Anyone can create a token with any payload they like. A token is only trustworthy when its signature checks out against the right key. If verification passes, whoever holds that key signed it, and it hasn't been changed since.

The tool also refuses to "verify" an unsigned token. If the header says `alg: none`, it tells you the token is unsigned, so anyone could have written it.

## Tips and common problems

**"Expired" but you just got the token.** The page compares `exp` with your device's clock. Check that your clock is correct, then check whether the issuer uses a very short lifetime.

**Verification fails with the right secret.** Many systems store HMAC secrets Base64-encoded. Try ticking **The secret is Base64-encoded**. Also make sure there's no trailing space or newline.

**Verification fails with a public key.** Paste the full PEM, including the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` lines, and make sure it matches the algorithm in the header (an EC key for ES256, an RSA key for RS256 or PS256).

**Verifying APNs or Firebase tokens.** APNs provider tokens use ES256, so paste the matching public key. If you need to *create* an APNs token from a .p8 key, use the [APNs JWT generator](https://bestlatestapps.com/tools/apns-jwt-generator/).

**Payload isn't JSON.** If a part can't be decoded, the token was probably truncated when copied. Copy it again, all three parts.

## FAQ

**Is it safe to paste a token here?**
It's decoded locally, and the page doesn't put it in the URL, in storage or in any network request. Even so, use test or expired tokens where you can.

**Does decoding prove a token is real?**
No. Only a successful signature check with the right key does.

**Which algorithms can it verify?**
HS256/384/512 with a secret, and RS256, PS256 and ES256 (and larger variants) with a PEM public key.

## Try it

Next time a token misbehaves: **[JWT decoder, free, in your browser](https://bestlatestapps.com/tools/jwt-decoder/)**. Related tools: [JSON formatter](https://bestlatestapps.com/tools/json-formatter/) and [Base64](https://bestlatestapps.com/tools/base64/).

*— Alok Kumar Singh*
{% endraw %}
