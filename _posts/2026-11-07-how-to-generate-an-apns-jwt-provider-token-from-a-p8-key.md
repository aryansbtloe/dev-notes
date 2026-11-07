---
layout: post
title: "How to Generate an APNs JWT Provider Token From a .p8 Key"
date: 2026-11-07 10:00:00 +0530
author: Alok Singh
description: "Sign an APNs provider token (ES256 JWT) from your .p8 key in the browser, test it with a ready curl command, and avoid the signature format and refresh-timing traps."
canonical_url: "https://bestlatestapps.com/blog/how-to-generate-an-apns-jwt-provider-token-from-a-p8-key/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-generate-an-apns-jwt-provider-token-from-a-p8-key/).

{% raw %}
Token-based push authentication is simpler than certificates once it works. Getting it to work the first time, though, usually involves a 403 InvalidProviderToken with no further explanation. [APNs JWT generator](https://bestlatestapps.com/tools/apns-jwt-generator/) builds and signs a correct token from your .p8 key, Key ID and Team ID, decodes it, counts down to expiry, and gives you a curl command to test against APNs.

## How the token is built

An APNs provider token is a JWT:

- **Header:** `{"alg": "ES256", "kid": "<Key ID>"}`
- **Claims:** `{"iss": "<Team ID>", "iat": <Unix seconds>}`

Each part is base64url-encoded without padding, joined with a dot, and signed using ECDSA on the P-256 curve with SHA-256. The base64url signature becomes the third part. The page does the signing with the browser's Web Crypto API and shows the decoded header and claims next to the result.

## Where the IDs come from

- **Key ID:** in Certificates, Identifiers & Profiles → Keys, and in the file name `AuthKey_XXXXXXXXXX.p8` (the page reads it from the name when it can).
- **Team ID:** in your developer account's Membership details. Ten characters. It's the `iss` claim, **not** your bundle ID.
- **apns-topic:** your bundle ID, sent as a header. The token isn't tied to one app; the topic picks the app per request.

## The signature format trap

If your server's tokens are rejected but this page's work, compare the lengths. In the JWT world, an ES256 signature is a fixed 64 bytes, the two 32-byte numbers `r` and `s` back to back. OpenSSL, Java's `Signature` class and plenty of older libraries return a DER-encoded signature instead, usually 70–72 bytes, and APNs rejects it with 403 InvalidProviderToken. Unwrap the DER structure into the two raw numbers, pad each to 32 bytes on the left, concatenate them, and the error goes away.

## The 20-minute and 60-minute rules

- APNs rejects a token whose `iat` is more than an hour old: **403 ExpiredProviderToken**. The page's countdown shows how long is left.
- Refreshing too often is also a problem: generating new tokens more than once every 20 minutes can earn **429 TooManyProviderTokenUpdates**.
- So keep a single token per key in memory and swap in a fresh one once it's between 20 and 60 minutes old.
- Use your server's clock in UTC; a skewed clock shifts `iat`.

There's a **Set 61 min ago** option that builds an already-expired token, so you can test how your server handles ExpiredProviderToken.

## Testing it

Copy the curl command, replace `com.example.app` with your bundle ID, and set `DEVICE_TOKEN` to a device token from a build whose signing environment matches the endpoint (sandbox vs production mismatches are a classic). A 200 means APNs accepted the push. Anything else comes with a reason string; the site's [APNs error codes](https://bestlatestapps.com/tools/apns-error-codes/) page explains them.

## Token vs certificate auth

Token auth is easier to run: one .p8 key covers every app in your team, works for sandbox and production, and doesn't expire yearly the way push certificates do. Certificate auth still works and suits tools built around a certificate in the keychain.

## About the key

The page says the .p8 is read and used only in the tab via Web Crypto, isn't sent anywhere and isn't stored. That's fine for testing. In production, tokens should be signed server-side with the key held in a secrets manager, not on developer laptops. The page shows Google ads, like the rest of the site.

## FAQ

**How long is a token valid?**
Up to an hour from `iat`; refresh no more often than every 20 minutes.

**InvalidProviderToken vs ExpiredProviderToken?**
Expired means valid signature but older than an hour. Invalid means APNs couldn't verify it: a mismatched Key ID or Team ID, a key that's been revoked, or a DER-encoded signature.

**Can one key serve all my apps?**
Yes. A key scoped to your team signs for all its apps, sandbox and production alike.

**Is my key uploaded?**
No, according to the page.

Next time APNs says 403, generate a known-good token with **[APNs JWT generator](https://bestlatestapps.com/tools/apns-jwt-generator/)**.

*— Alok Kumar Singh*
{% endraw %}
