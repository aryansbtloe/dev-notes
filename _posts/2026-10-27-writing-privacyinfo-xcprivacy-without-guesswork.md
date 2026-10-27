---
layout: post
title: "Writing PrivacyInfo.xcprivacy Without Guesswork: How My Browser-Based Privacy Manifest Generator Models Apple's Rules"
date: 2026-10-27 10:00:00 +0530
author: Alok Singh
description: "The four keys, the seventeen required-reason codes, the consistency checks that catch half-finished manifests, and why round-tripping an existing file matters more than generating a new one"
canonical_url: "https://bestlatestapps.com/blog/writing-privacyinfo-xcprivacy-without-guesswork/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/writing-privacyinfo-xcprivacy-without-guesswork/).

{% raw %}
### The four keys, the seventeen required-reason codes, the consistency checks that catch half-finished manifests, and why round-tripping an existing file matters more than generating a new one

Since 2024, Apple has asked apps and third-party SDKs to describe their data use in a privacy manifest: a property list called `PrivacyInfo.xcprivacy`. Apps that use certain system APIs, the "required-reason APIs", have to declare why. Most iOS apps hit at least one, because `UserDefaults` is on the list.

Xcode has a template for the file and a property-list editor, but the editor shows raw keys and makes you look up reason codes like `CA92.1` in the documentation. I built a [privacy manifest generator](https://bestlatestapps.com/tools/privacy-manifest-generator/) for my site so the choices are checkboxes with plain descriptions, and the output is a file you can drop into a target.

This article explains how the tool models the manifest and what it checks, using its real source, `privacy-manifest.js`. Snippets are taken or trimmed from that file. I also checked the tool's list of reason codes against Apple's documentation while writing; more on that below. What the tool can't do for you, which is know what your binary actually calls, gets its own section.

------------------------------------------------------------------------

## 1. What a privacy manifest contains

A privacy manifest is a dictionary with four top-level keys:

| Key                           | Type                  | What it says                                                                                                         |
|-------------------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------|
| `NSPrivacyTracking`           | Boolean               | Whether the app or SDK uses data for tracking, as Apple's App Tracking Transparency rules define it.                 |
| `NSPrivacyTrackingDomains`    | Array of strings      | Domains contacted for tracking. iOS can block them until the person allows tracking.                                 |
| `NSPrivacyCollectedDataTypes` | Array of dictionaries | Each data type collected, whether it's linked to the person, whether it's used for tracking, and why it's collected. |
| `NSPrivacyAccessedAPITypes`   | Array of dictionaries | Each required-reason API category used, with one or more approved reason codes.                                      |

When you archive, Xcode combines the manifests of your app and its SDKs into a privacy report. The data types section should line up with the privacy "nutrition label" you fill in on App Store Connect. That's the real value of getting the file right: it's one source of truth that the report, the label and App Review can all be checked against.

------------------------------------------------------------------------

## 2. Modelling Apple's vocabulary as data

The tool's first design decision is that Apple's vocabulary lives in three plain arrays at the top of the file, not scattered through UI code. Collected data types are grouped the way Apple groups them:

``` javascript
var TYPES = [
  ["Contact info", [["Name", "Name"], ["EmailAddress", "Email address"], ["PhoneNumber", "Phone number"],
                    ["PhysicalAddress", "Physical address"], ["OtherUserContactInfo", "Other contact info"]]],
  ["Health and fitness", [["Health", "Health"], ["Fitness", "Fitness"]]],
  // ... Financial info, Location, Sensitive info, Contacts, User content, Browsing and search,
  //     Identifiers, Purchases, Usage data, Diagnostics, Surroundings and body, Other
];
var PURPOSES = [["ThirdPartyAdvertising", "Third-party advertising"], ["DeveloperAdvertising", "Developer's advertising"],
  ["Analytics", "Analytics"], ["ProductPersonalization", "Product personalisation"],
  ["AppFunctionality", "App functionality"], ["Other", "Other purposes"]];
```

Each entry pairs Apple's key suffix with a human label. The suffix is exact, including Apple's own oddities: the photos type is `PhotosorVideos`, with a lower-case "or", and the generator keeps it that way because the string has to match.

Required-reason APIs are modelled the same way, as category, description, and a list of codes with what each one permits:

``` javascript
var APIS = [
  ["UserDefaults", "User defaults (UserDefaults, NSUserDefaults, @AppStorage)", [
    ["CA92.1", "Read and write data that only this app can access."],
    ["1C8F.1", "Read and write data shared with apps, extensions and App Clips in the same App Group."],
    ["C56D.1", "Third-party SDK: a wrapper the app calls, using only data the app passes in."],
    ["AC6B.1", "Read the MDM managed app configuration, or store feedback for MDM to query."]]],
  ["FileTimestamp", "File timestamps (creationDate, modificationDate, stat, getattrlist)", [ /* C617.1, DDA9.1, 3B52.1, 0A2A.1 */ ]],
  ["SystemBootTime", "System boot time (systemUptime, mach_absolute_time)", [ /* 35F9.1, 8FFB.1, 3D61.1 */ ]],
  ["DiskSpace", "Disk space (volumeAvailableCapacity, statfs)", [ /* E174.1, 85F4.1, 7D9E.1, B728.1 */ ]],
  ["ActiveKeyboards", "Active keyboards (activeInputModes)", [ /* 3EC4.1, 54BD.1 */ ]]
];
```

That's five categories and seventeen codes. While writing this I compared them with Apple's current documentation for `NSPrivacyAccessedAPIType`: the same five categories and the same seventeen codes. Apple can add codes, so if you're reading this much later, the documentation wins over any tool, including mine.

Why does "data, not code" matter here? Because this list changes on Apple's schedule, not mine. Updating it means editing one array, and the checkboxes, the generated XML and the import logic all follow, since they all read the same table.

------------------------------------------------------------------------

## 3. Generating the plist by hand

The output is built line by line rather than with an XML library. That sounds primitive, but a property list is a tiny, fixed grammar, and writing it directly gives full control over the formatting Xcode users expect: tabs, Apple's DOCTYPE, and `<array/>` for empty arrays.

``` javascript
function plist(st) {
  var t = "\t", L = ['<?xml version="1.0" encoding="UTF-8"?>',
    '<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">',
    '<plist version="1.0">', "<dict>"];
  L.push(t + "<key>NSPrivacyTracking</key>", t + bool(st.tracking));
  L.push(t + "<key>NSPrivacyTrackingDomains</key>");
  if (st.domains.length) { L.push(t + "<array>"); st.domains.forEach(function (d) { L.push(t + t + "<string>" + x(d) + "</string>"); }); L.push(t + "</array>"); }
  else L.push(t + "<array/>");
  // ... NSPrivacyCollectedDataTypes and NSPrivacyAccessedAPITypes in the same style
  L.push("</dict>", "</plist>", "");
  return L.join("\n");
}
```

Every user-supplied string goes through `x()`, which escapes `&`, `<` and `>`. In practice the only free text is the tracking domains, but a manifest that fails to parse because a domain field held a stray character is precisely the kind of bug that shows up at archive time, the worst possible moment.

Key names are composed from a prefix: `NSPrivacyCollectedDataType` + `EmailAddress`, `NSPrivacyCollectedDataTypePurpose` + `Analytics`, `NSPrivacyAccessedAPICategory` + `UserDefaults`. Here's real output for an app that collects crash data for app functionality and uses UserDefaults and file timestamps (I ran the function in Node to get it):

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeCrashData</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

Note that all four keys are always written, even when empty. A manifest that explicitly says tracking is off, with no domains and no collected data, is easier to review than one that leaves the reader wondering whether a key was forgotten.

------------------------------------------------------------------------

## 4. Catching half-finished manifests

The most common mistakes in a privacy manifest aren't syntax errors. They're contradictions: tracking off, but a data type marked as used for tracking; tracking domains listed, but tracking off; a data type collected for no stated purpose. The `warnings()` function checks exactly those:

``` javascript
function warnings(st) {
  var w = [];
  if (st.tracking && !st.domains.length) w.push("Tracking is on. List the domains your app or SDKs use for tracking, so iOS can block them until the person allows tracking.");
  if (!st.tracking && st.domains.length) w.push("Tracking domains are listed while tracking is off. Turn tracking on, or clear the domains.");
  Object.keys(st.types).forEach(function (k) {
    var d = st.types[k];
    if (!d.purposes.length) w.push(k + ": choose at least one purpose.");
    if (d.tracking && !st.tracking) w.push(k + " is marked as used for tracking, so turn on tracking at the top too.");
  });
  return w;
}
```

The UI also prevents two of these before they happen:

- **Ticking a data type pre-ticks "App functionality"** if no purpose is chosen yet. Almost every collected type has that purpose, and it means a freshly ticked type is never purposeless.
- **Ticking "Used for tracking" on any data type turns on the top-level tracking switch.** The two can't disagree in that direction.

The status line underneath summarises the state, for example "1 data type collected · 2 required-reason API categories declared", so you can sanity-check the result at a glance before downloading.

Tracking domains get light normalisation as you type: the field is split on whitespace and commas, and any `http://` or `https://` prefix and any path are stripped, because the key expects domains, not URLs. Pasting `https://ads.example.com/sdk/v2` gives `ads.example.com`.

------------------------------------------------------------------------

## 5. Round-tripping an existing manifest

Most of the time you're not writing a manifest from nothing. You have one from last year, or one an SDK vendor shipped, and you need to change it. So the tool can open an existing `.xcprivacy` file, tick the matching boxes, and write it back out.

Reading uses the browser's `DOMParser` and a short recursive reader for the plist types. The interesting part is what happens to things the page has no checkbox for:

``` javascript
(o.NSPrivacyAccessedAPITypes || []).forEach(function (d) {
  var c = String(d.NSPrivacyAccessedAPIType || "").replace("NSPrivacyAccessedAPICategory", "");
  if (!c) return;
  var known = APIS.filter(function (a) { return a[0] === c; })[0], codes = (d.NSPrivacyAccessedAPITypeReasons || []).map(String);
  st.apis[c] = codes;
  codes.forEach(function (r) { if (!known || !known[2].some(function (k) { return k[0] === r; })) unknown.push(c + " " + r); });
});
```

Unknown codes, perhaps a code Apple added after the tool's table was written, aren't dropped. A separate `extras()` function collects them, and `read()` merges them back into the state every time the output is regenerated. The status line tells you which entries were kept as they were. Silently deleting a reason code you don't recognise would be the worst possible behaviour for an editing tool, because the file would still be valid and the loss would be invisible.

The file input is cleared as soon as the file is read, and a file that isn't a readable XML property list gets a plain error ("This file is unreadable as an XML property list") instead of a half-filled form.

------------------------------------------------------------------------

## 6. Defaults and persistence

On a first visit, the form starts with one thing ticked: `UserDefaults` with reason `CA92.1`, "read and write data that only this app can access". That's not a guess about your app. It's a starting point that matches the most common case, and you untick it if it doesn't apply. The page FAQ makes the related point: if you use any required-reason API, you need a manifest even when you collect no data.

Your choices are saved in the browser's `localStorage` under one key, so they're there next time you open the page. The page says so in its privacy note, and the save is wrapped in a `try/catch` so private-browsing modes that refuse storage don't break the tool. The manifest is built inside the page from what you tick; the tool's code doesn't send your choices anywhere. Download produces a `Blob` named `PrivacyInfo.xcprivacy`, with a reminder to add it to your app target in Xcode.

------------------------------------------------------------------------

## 7. What a generator can't know

This is the section I'd want every privacy-manifest tool to include. A form can make the file correct. It can't make it *true*. Only you know what your binary and its dependencies do.

Here's how I'd approach the audit. It's general advice, not something the tool does for you:

- **Search your own code for the API families.** `UserDefaults` and `@AppStorage`; `creationDate`, `modificationDate` and `stat`; `systemUptime` and `mach_absolute_time()`; `volumeAvailableCapacity` keys; `activeInputModes`. The tool's category descriptions name the common entry points, which makes a good search list.
- **Remember property wrappers and helpers.** `@AppStorage` is UserDefaults. A "time since launch" helper is probably system boot time. Measuring intervals with `mach_absolute_time` falls under `35F9.1`.
- **Collect your SDKs' manifests.** Third-party SDKs are expected to ship their own `PrivacyInfo.xcprivacy`. Your app's manifest covers your code; theirs covers theirs. Update SDKs that don't ship one yet.
- **Generate Xcode's privacy report** from an archive and read it as a whole. It's the combined view, and it's what should match your App Store Connect privacy label.
- **Revisit it with every new dependency.** Analytics, crash reporting and ad SDKs are the usual sources of new data types.

If you also ship ads, two related tools on my site help with the rest of that paperwork: the [app-ads.txt generator and checker](https://bestlatestapps.com/tools/app-ads-txt/) and the [eCPM calculator](https://bestlatestapps.com/tools/ecpm-calculator/).

------------------------------------------------------------------------

## 8. Picking reasons: the cases people ask about

The reason codes are terse, so here's how the tool's descriptions map to everyday situations. The final call is yours, checked against Apple's documentation, but these are the common situations:

| Situation                                                           | Category         | Code     |
|---------------------------------------------------------------------|------------------|----------|
| Settings and flags stored by your app only                          | UserDefaults     | `CA92.1` |
| A widget or extension reads the app's settings through an App Group | UserDefaults     | `1C8F.1` |
| You're an SDK wrapping UserDefaults for the host app                | UserDefaults     | `C56D.1` |
| Checking the age or size of your own cache files                    | File timestamps  | `C617.1` |
| Showing "modified" dates in a file browser                          | File timestamps  | `DDA9.1` |
| Timing events or running timers inside the app                      | System boot time | `35F9.1` |
| Checking free space before writing a large download                 | Disk space       | `E174.1` |
| A custom keyboard app checking which keyboards are active           | Active keyboards | `3EC4.1` |

Multiple reasons per category are allowed and common. An app that stores its own settings *and* shares some with a widget declares both `CA92.1` and `1C8F.1`.

------------------------------------------------------------------------

## 9. Before you submit: a privacy manifest checklist

1.  **All four keys are present**, empty arrays included.
2.  **Every required-reason API you call is declared**, with every reason that applies.
3.  **Every collected data type has at least one purpose**, and linked or tracking flags that match reality.
4.  **Tracking and tracking domains agree**: both on, or both off.
5.  **Your SDKs ship their own manifests**, and you've updated the ones that didn't.
6.  **Xcode's privacy report matches your App Store Connect label.**
7.  **Unknown or newer codes in an old file were kept**, not dropped, when you edited it.
8.  **You re-checked Apple's current list** of categories and codes.

------------------------------------------------------------------------

## Wrapping up

A privacy manifest is a small file with a large blast radius. The tool's job is to make the format impossible to get wrong: Apple's vocabulary as data, hand-written XML with escaping, consistency warnings, and an import path that never throws away what it doesn't understand. Your job is the part no tool can do: knowing what your app and its SDKs actually do.

Try it, or browse the related tools:

- **Privacy manifest generator:** https://bestlatestapps.com/tools/privacy-manifest-generator/
- **.mobileprovision viewer:** https://bestlatestapps.com/tools/mobileprovision-viewer/
- **App Store character counter:** https://bestlatestapps.com/tools/app-store-character-counter/
- **All of my free web tools:** https://bestlatestapps.com/tools/
- **My portfolio:** https://bestlatestapps.com/alok/

Audit your dependencies as well as your own code: an SDK's missing manifest is still your rejection email.

*— Alok Kumar Singh*
{% endraw %}
