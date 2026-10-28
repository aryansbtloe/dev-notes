---
layout: post
title: "Why Your Universal Links Open Safari: Validating the apple-app-site-association File"
date: 2026-10-28 10:00:00 +0530
author: Alok Singh
description: "Validate an apple-app-site-association file, test which URLs open your iOS app, and fix the hosting and CDN problems that silently break universal links."
canonical_url: "https://bestlatestapps.com/blog/why-your-universal-links-open-safari-validating-the-apple-app/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/why-your-universal-links-open-safari-validating-the-apple-app/).

{% raw %}
Universal links are one of those iOS features that either work perfectly or fail silently. Tap a link, Safari opens instead of your app, and there's no error anywhere. An afternoon can disappear into something as small as a missing Team ID prefix or a redirect from the bare domain to www. [AASA validator](https://bestlatestapps.com/tools/aasa-validator/) checks the file's structure, explains each rule, tests URLs against your rules, and walks through the hosting checklist.

## What it checks in the file

- **Valid JSON** (no comments, no trailing commas, no byte-order mark, no curly quotes pasted from a doc).
- **Size:** at most 128 KB uncompressed, Apple's limit.
- **App IDs** written as `<Team ID>.<bundle ID>`. The prefix is almost always your 10-character uppercase Team ID. A bare bundle ID never matches; this is the single most common mistake.
- **components** entries only use keys Apple defines (`/`, `?`, `#`, `exclude`, `comment`, `caseSensitive`, `percentEncoded`), and `exclude` is a real boolean, not the string `"true"`.
- Structural problems such as an app listed in two `details` entries, empty `apps` arrays in `webcredentials` or `appclips`, or a catch-all `{"/": "*"}` that makes every rule after it unreachable.
- **A signed (PKCS#7) file**, which iOS hasn't accepted since iOS 9.
- **HTML instead of JSON**, the telltale sign your server returned a 404, a login page or a redirect page.

## Testing URLs

Paste URLs and the tool shows which rule matches first and whether your app would open. The model follows Apple's documented behaviour: for each app, iOS walks `components` from top to bottom and stops at the first entry whose path, query and fragment patterns all match. If that entry has `"exclude": true`, the browser opens; otherwise the app does. No match means the browser.

Pattern syntax: `*` matches any run of characters, `?` exactly one, and `?*` at least one. A `?` key holding an object matches named query parameters.

This is a model, not a device test. Confirm on a real device under Settings → Developer → Universal Links → Diagnostics (with Developer Mode on).

## Hosting: where most failures really are

The file has to be served:

- at `https://<domain>/.well-known/apple-app-site-association`, **no extension**;
- over HTTPS with a valid certificate;
- with **status 200 and no redirect**, on every domain and subdomain you list (Apple doesn't follow redirects, so `example.com` → `www.example.com` breaks it);
- as `application/json`, unsigned;
- with no login, firewall rule or bot protection in front of it.

That last one bites people using aggressive bot protection: Apple's servers get challenged and never see the file. The page gives you curl commands to check both your server and what Apple's CDN currently holds.

## Apple's CDN

Since iOS 14 and macOS 11, devices fetch the file from Apple's CDN, not your server. The CDN picks up changes within about 24 hours; devices fetch it at install and check roughly weekly. There's no purge button. On a development device, `?mode=developer` in the associated domain entitlement bypasses the CDN while you test.

## components or paths?

Use `components` for iOS 13 and later; it matches path, query and fragment separately and makes exclusions explicit. The older `paths` array with `NOT` prefixes still works. Pick one format per `details` entry.

## Privacy

The page says the checks run in your browser and its code doesn't send your file or domain anywhere. Like the rest of the site, the page shows Google ads.

## FAQ

**Why do my links open Safari?**
Wrong or missing Team ID prefix, a redirect or bot protection on the file, a stale CDN copy, an earlier rule or exclude matching first, or typing the URL straight into Safari's address bar (which never opens the app).

**Does the file need a .json extension?**
No. No extension, served as JSON.

**How do I clear Apple's CDN?**
You can't. Wait, or use developer mode on a test device.

**Is the URL test definitive?**
No, it's a model of Apple's rules. Check Diagnostics on a device.

Before you blame iOS, run your file through **[AASA validator](https://bestlatestapps.com/tools/aasa-validator/)**.

*— Alok Kumar Singh*
{% endraw %}
