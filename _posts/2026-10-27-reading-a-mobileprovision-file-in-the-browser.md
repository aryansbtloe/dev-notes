---
layout: post
title: "Reading a .mobileprovision File in the Browser: Finding the Plist, Parsing It Without a DOM, and Decoding Certificates by Hand"
date: 2026-10-27 10:00:00 +0530
author: Alok Singh
description: "How my provisioning profile viewer pulls the XML out of a signed wrapper, reads X.509 certificates with a tiny DER reader, infers the profile type and explains the failures it finds"
canonical_url: "https://bestlatestapps.com/blog/reading-a-mobileprovision-file-in-the-browser/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/reading-a-mobileprovision-file-in-the-browser/).

{% raw %}
### How my provisioning profile viewer pulls the XML out of a signed wrapper, reads X.509 certificates with a tiny DER reader, infers the profile type and explains the failures it finds

Every iOS developer has had this afternoon. A build installs on your phone and not on your tester's. Push works in debug and fails in TestFlight. An extension can't reach the App Group. Nine times out of ten, the answer is sitting in a provisioning profile that you can't easily read.

On a Mac, the classic move is `security cms -D -i profile.mobileprovision` and a scroll through a wall of XML. That works if you're on a Mac, you remember the flags, and you know what to look for. I wanted something quicker: drop the file on a web page and get the answers that matter (type, expiry, devices, certificates, entitlements), with warnings in plain English.

So I built the [.mobileprovision viewer](https://bestlatestapps.com/tools/mobileprovision-viewer/) on my site. This article walks through how it works, using the tool's real source. The snippets are taken from, or trimmed down from, `mobileprovision.js`. Where I generalise beyond what the code does, I say so.

------------------------------------------------------------------------

## 1. What is actually inside a provisioning profile

A `.mobileprovision` (or `.provisionprofile` on the Mac) is a **property list wrapped in a CMS signature**. CMS is the Cryptographic Message Syntax, the standard that grew out of PKCS \#7. Apple signs the plist, and the signed bundle is stored in DER, the binary encoding used for certificates.

Two facts shape the whole design:

1.  **The format isn't documented as an API.** Apple's technote TN3125 describes profiles but is clear that the file's layout can change, and that modern systems rely on a binary `DER-Encoded-Profile` entry inside the plist as the source of truth. A viewer should read the human-readable plist and not pretend to be the system's validator.
2.  **CMS signed data with embedded content stores the content as-is.** The XML plist sits inside the DER bytes unencrypted. You don't need a CMS parser to find it; you need to find the bytes between `<?xml` and `</plist>`.

That second fact is the shortcut the whole tool is built on. It also sets the limit I'm honest about on the page: the viewer reads a profile, it doesn't prove that Apple signed it. Every result says "Signature not verified".

------------------------------------------------------------------------

## 2. Finding the plist without a CMS parser

The file arrives as an `ArrayBuffer` from `FileReader`. The first step converts the bytes to a "binary string", one character per byte, so ordinary string searches work:

``` javascript
function bytesToLatin1(u8) {
  var s = "", CH = 0x8000;
  for (var i = 0; i < u8.length; i += CH)
    s += String.fromCharCode.apply(null, u8.subarray(i, i + CH));
  return s;
}
```

The chunking matters. `String.fromCharCode.apply` passes every byte as a separate function argument, and engines cap how many arguments a call can take. A profile with a few hundred device UDIDs and several certificates is easily tens of kilobytes, so the conversion runs in 32 KB slices.

Then the extraction itself:

``` javascript
function extractPlist(u8) {
  var s = bytesToLatin1(u8), a = s.indexOf("<?xml"), b = s.indexOf("</plist>", a);
  if (a < 0) a = s.indexOf("<plist");
  if (a < 0 || b < 0) {
    if (s.slice(0, 6) === "bplist")
      throw new Error("This is a binary plist, not a provisioning profile. Convert it with: plutil -convert xml1 file");
    throw new Error("No property list found. Is this a .mobileprovision or .provisionprofile file?");
  }
  return utf8(u8.subarray(a, b + 8));
}
```

Three details I like here:

- **The search is on Latin-1, the decode is UTF-8.** Indexes found in the one-byte-per-character string are byte offsets, so slicing the original `Uint8Array` with them is exact. Only that slice is decoded as UTF-8, so a team name with non-ASCII characters comes out right.
- **A fallback for a missing XML declaration.** If there's no `<?xml`, it looks for `<plist` directly.
- **An error message that tells you what to do.** People do drop random `.plist` files on a profile viewer. A binary plist starts with the magic `bplist`, so the tool says exactly that and gives the `plutil` command to convert it.

------------------------------------------------------------------------

## 3. A minimal plist parser that doesn't need a DOM

Browsers have `DOMParser`, and my [privacy manifest generator](https://bestlatestapps.com/tools/privacy-manifest-generator/) uses it. For this tool I wanted the parsing code to run in Node too, so the logic could be tested without a browser. The module ends with the usual dual export:

``` javascript
var api = { extractPlist: extractPlist, parsePlist: parsePlist, analyze: analyze, cert: cert, profileType: profileType, fmtVal: fmtVal };
if (typeof module === "object" && module.exports) module.exports = api;
root.MobileProvision = api;
if (typeof document === "undefined") return;
```

Everything above that last line is pure. Everything below it wires up the page. That split is the most reusable idea in this article: keep the parsing in functions that take bytes and return data, and keep the DOM at the edge.

The parser itself is a tokenizer plus a recursive reader. One regular expression finds tags, comments, processing instructions and the DOCTYPE. Each tag token remembers the text that came *before* it:

``` javascript
var re = /<(\/?)([a-zA-Z]+)(?:\s+[a-zA-Z][^>]*?)?(\s*\/)?>|<!--[\s\S]*?-->|<\?[\s\S]*?\?>|<!DOCTYPE[^>]*>/g;
while ((m = re.exec(xml))) {
  if (m[2]) toks.push({ close: !!m[1], tag: m[2], self: !!m[3], start: m.index, end: re.lastIndex,
                        textBefore: xml.slice(last, m.index) });
  last = re.lastIndex;
}
```

Storing `textBefore` on each token is a neat trick. For `<string>Demo</string>`, the closing tag's `textBefore` is exactly the string value. The reader never has to track character positions; it asks the closing token for its text.

The value reader is a `switch` over the plist types. Trimmed:

``` javascript
switch (t.tag) {
  case "dict":
    var o = {};
    while (toks[i] && !(toks[i].close && toks[i].tag === "dict")) {
      var k = toks[i++]; if (k.tag !== "key") throw new Error("Expected <key> in <dict>");
      var ke = expect("key"); o[decodeEntities(ke.textBefore)] = val();
    }
    expect("dict"); return o;
  case "array": /* same idea, collecting val() until </array> */
  case "true": expect("true"); return true;
  case "false": expect("false"); return false;
  default:
    var e = expect(t.tag), text = decodeEntities(e.textBefore);
    if (t.tag === "string") return text;
    if (t.tag === "integer") return parseInt(text, 10);
    if (t.tag === "real") return parseFloat(text);
    if (t.tag === "date") return new Date(text.trim());
    if (t.tag === "data") return b64(text);
    return text;
}
```

Self-closing tags such as `<true/>`, `<array/>` and `<dict/>` are handled before the `switch`, and `decodeEntities` handles the five named XML entities plus decimal and hex character references. `<data>` becomes a `Uint8Array`, which matters, because that's where the certificates live.

Is this a complete XML parser? No, and it doesn't need to be. Apple writes these plists in one predictable shape. A malformed file throws "Malformed plist near \<tag\>" rather than being guessed at, which is the right failure for a diagnostic tool.

------------------------------------------------------------------------

## 4. Decoding the certificates with a tiny DER reader

The `DeveloperCertificates` array holds X.509 certificates as DER bytes. "Which certificate is in this profile, and has it expired?" is one of the most useful questions the tool answers, so it needs to read a certificate's subject, issuer, validity dates and serial number. Pulling in a full ASN.1 library for that felt heavy, so the tool has a reader of about 30 lines.

DER is tag, length, value. The length is either one byte, or a byte with the high bit set that says how many length bytes follow:

``` javascript
function der(u8, p) {
  var tag = u8[p], len = u8[p + 1], hl = 2;
  if (len & 0x80) { var n = len & 0x7f; len = 0; for (var k = 0; k < n; k++) len = len * 256 + u8[p + 2 + k]; hl = 2 + n; }
  return { tag: tag, start: p, hs: p + hl, end: p + hl + len };
}
function kids(u8, node) { var out = [], p = node.hs; while (p < node.end) { var c = der(u8, p); out.push(c); p = c.end; } return out; }
```

With `der` and `kids` you can walk any DER structure. A certificate is a SEQUENCE whose first child is the "to be signed" part. Inside that, an optional explicit version tag `[0]` (byte `0xa0`) may come first, so the code checks for it and shifts its field indexes:

``` javascript
var c = der(u8, 0), tbs = kids(u8, c)[0], f = kids(u8, tbs), i = f[0].tag === 0xa0 ? 1 : 0;
// f[i] serial, f[i+1] signature algorithm, f[i+2] issuer, f[i+3] validity, f[i+4] subject
```

The remaining helpers each handle one small piece of the encoding:

- **Object identifiers.** The first byte packs two numbers (`b[0] / 40` and `b[0] % 40`); the rest are base-128 with a continuation bit. The tool maps four of them: `2.5.4.3` (CN), `2.5.4.10` (O), `2.5.4.11` (OU) and `2.5.4.6` (C). For an Apple signing certificate, CN is the "Apple Development: Name (ID)" line and OU is the Team ID.
- **Strings.** Most are UTF-8 or printable strings, decoded as UTF-8. Tag `0x1e`, a BMPString, is two bytes per character big-endian, so it gets its own branch.
- **Times.** X.509 uses UTCTime (tag `0x17`, two-digit year) up to 2049 and GeneralizedTime (four-digit year) after. The code applies the standard pivot: two-digit years under 50 are 20xx, the rest 19xx.
- **Serial numbers.** Printed as upper-case hex with a single leading `00` stripped. DER adds that byte when the top bit of the first real byte is set, so the number isn't read as negative, and it isn't part of the number you'd see in Keychain Access.

The whole `cert()` function is wrapped in `try/catch`. A certificate this reader can't decode becomes a row that says so and points to Keychain Access, rather than crashing the page. The warnings then only count certificates that *were* readable when deciding whether "every certificate has expired". A comment in the source calls that out, and it's the right call: a parse failure is not evidence of expiry.

To be clear about scope: this reads certificate fields, it doesn't build a chain or check a signature. That's the same "viewing only" line the page draws everywhere.

------------------------------------------------------------------------

## 5. Inferring the profile type

A profile doesn't contain a "type" field. Xcode and the developer portal show one, but in the file you infer it from three things: whether it lists devices, whether it provisions all devices, and the `get-task-allow` entitlement, which lets a debugger attach.

``` javascript
function profileType(p) {
  var ent = p.Entitlements || {}, gta = ent["get-task-allow"] === true || ent["com.apple.security.get-task-allow"] === true;
  var mac = (p.Platform || []).indexOf("OSX") >= 0;
  if (p.ProvisionsAllDevices) return mac ? "Developer ID" : "Enterprise (In-House)";
  if (p.ProvisionedDevices) return gta ? "Development" : "Ad Hoc";
  return gta ? "Development (no device list)" : mac ? "Mac App Store" : "App Store";
}
```

As a table:

| Device list            | get-task-allow   | Reported type                                     |
|------------------------|------------------|---------------------------------------------------|
| `ProvisionsAllDevices` | any              | Enterprise (In-House), or Developer ID on the Mac |
| `ProvisionedDevices`   | true             | Development                                       |
| `ProvisionedDevices`   | false or missing | Ad Hoc                                            |
| none                   | false or missing | App Store (or Mac App Store)                      |

The macOS entitlement name `com.apple.security.get-task-allow` is checked alongside the iOS one, so Mac profiles are classified too. The UI labels the row "Type (inferred)" on purpose. It's a well-founded rule, not a field Apple wrote down, and I'd rather the label admit that.

------------------------------------------------------------------------

## 6. Turning fields into warnings that explain real failures

A table of keys and values is only half a tool. The `analyze()` function also produces warnings, each tagged `err`, `warn` or `note`:

``` javascript
if (exp && exp < n) warnings.push(["err", "This profile expired on " + exp.toISOString().slice(0, 10) + ". Regenerate it in Certificates, Identifiers & Profiles (or let Xcode manage signing)."]);
else if (days !== null && days < 30) warnings.push(["warn", "Expires in " + days + " day" + (days === 1 ? "" : "s") + "."]);
// ...
var aps = ent["aps-environment"] || ent["com.apple.developer.aps-environment"];
if (aps === "development") warnings.push(["note", "aps-environment is development: device tokens from this build only work with api.sandbox.push.apple.com."]);
if (aps === "production") warnings.push(["note", "aps-environment is production: tokens work with api.push.apple.com, not the sandbox."]);
```

The `aps-environment` note is the one I find most useful. A device token belongs to one APNs environment. Send a sandbox token to the production host, or the other way round, and APNs rejects it, typically with `BadDeviceToken`. The profile tells you which environment a build's tokens belong to before you go digging in server logs. I wrote about the token side of this in [The device side of APNs](https://bestlatestapps.com/blog/the-device-side-of-apns/), and the [APNs error codes](https://bestlatestapps.com/tools/apns-error-codes/) page lists the other reasons.

Two smaller choices are worth copying:

- **The clock is injectable.** `analyze(u8, now)` takes an optional date, so "expires in 13 days" can be tested deterministically. I ran the module in Node against a synthetic profile while writing this article, and the expiry and certificate warnings came out exactly as the code reads.
- **The bundle ID is derived, not stored.** `application-identifier` is the Team ID, a dot, then the bundle ID (or a wildcard ending in `*`). The tool strips a leading ten-character team prefix with `/^[A-Z0-9]{10}\./` to show the bundle ID on its own row.

------------------------------------------------------------------------

## 7. Handling a sensitive file in a web page

A provisioning profile isn't a secret like a private key, but it does contain your Team ID, your testers' device UDIDs and your entitlements. Treating it carefully was a design goal, and a few lines in the page code do real work.

**The file is read with `FileReader` in the tab.** The tool's code makes no network request with it; there's no upload endpoint for it to call. That's why the page says "this tool's code doesn't send it anywhere". I phrase it that way deliberately: it's a statement about the tool's code, which I can stand behind, rather than a sweeping promise about everything else a browser page might load.

**The file input is cleared after reading.**

``` javascript
/* the profile must not stay in the file input after it is read (input.files is readable by any script on the page) */
function resetFile() { var fi = $("[data-file]"); if (fi) fi.value = ""; }
// ...called before reading, in onload and onerror, and on pagehide
window.addEventListener("pagehide", resetFile);
```

The comment says why. Once the bytes are parsed, there's no reason for a `File` object to sit in a DOM element where any script on the page could read it.

**"Clear" means clear.** The Clear button doesn't just hide the result panel. It empties every table, the device list, the warnings and the raw plist view, and resets the search box, so nothing from the profile remains in the page.

**Everything is rendered with `textContent`.** The values come from a file the user dropped, which could have been crafted. Building rows with `textContent` rather than `innerHTML` means a "profile name" containing markup is shown as text, never run. It's a habit worth having in any tool that renders file contents.

------------------------------------------------------------------------

## 8. Using it day to day

The workflows I reach for most:

- **"Why won't it install on this phone?"** Open the build's profile and search the Devices list for the tester's UDID. App Store profiles have no device list at all, and Enterprise profiles cover all devices, so the count line says which case you're in.
- **"What was this build actually signed with?"** An `.ipa` is a zip. Inside, the `.app` bundle carries `embedded.mobileprovision`. Drop that file in and you see the profile the build shipped with, not the one you think Xcode used.
- **"Did my regenerated profile pick up the change?"** A profile is a snapshot. Adding a device or a capability doesn't change copies you already downloaded. Open the old and new files and compare the UUID, the device count and the entitlements.
- **"Is associated-domains in there?"** If `com.apple.developer.associated-domains` doesn't list your domain, no `apple-app-site-association` file will make universal links work. When it does list it, the [AASA validator](https://bestlatestapps.com/tools/aasa-validator/) checks the other half.

The tool page also notes where Xcode keeps downloaded profiles: `~/Library/Developer/Xcode/UserData/Provisioning Profiles` on Xcode 16 and later, and `~/Library/MobileDevice/Provisioning Profiles` before that. Each file is named by its UUID, which the viewer shows, so you can match a file on disk to what you're looking at.

------------------------------------------------------------------------

## 9. Before you ship a file-inspection tool: a checklist

1.  **Parse in pure functions** that take bytes and return data, with the DOM wiring at the bottom, so you can test them in Node.
2.  **Look for the cheap path first.** Here, "the content is embedded as-is" meant no CMS parser at all.
3.  **Convert large byte arrays to strings in chunks.** `apply` has an argument limit.
4.  **Fail with instructions.** "This is a binary plist; run `plutil -convert xml1`" beats "parse error".
5.  **Label inferences as inferences**, like "Type (inferred)", and say clearly what you don't verify.
6.  **Don't count failures as facts.** An unreadable certificate isn't an expired one.
7.  **Make "now" a parameter** anywhere you compute days left.
8.  **Render file contents with `textContent`**, clear file inputs after reading, and make Clear really clear.

------------------------------------------------------------------------

## Wrapping up

A provisioning profile looks like a cryptographic black box, but for viewing purposes it's an XML plist in a signed envelope. Find the XML by its markers, parse it with a small tokenizer, read the certificates with a 30-line DER walker, and spend the rest of your effort on what developers need: plain-English warnings about expiry, certificates and the push environment.

Try it, or browse the related tools:

- **.mobileprovision viewer:** https://bestlatestapps.com/tools/mobileprovision-viewer/
- **AASA validator:** https://bestlatestapps.com/tools/aasa-validator/
- **Push notification tester:** https://bestlatestapps.com/tools/push-notification-tester/
- **All of my free web tools:** https://bestlatestapps.com/tools/
- **My portfolio:** https://bestlatestapps.com/alok/

The next time a build installs on one device and not another, open the profile before you open Xcode's settings.

*— Alok Kumar Singh*
{% endraw %}
