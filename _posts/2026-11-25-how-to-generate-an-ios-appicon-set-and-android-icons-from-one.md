---
layout: post
title: "How to Generate an iOS AppIcon Set (and Android Icons) From One Image"
date: 2026-11-25 18:00:00 +0530
author: Alok Singh
description: "Create every app icon size for iOS (including iOS 18 dark and tinted), macOS, watchOS, visionOS, Android adaptive and themed icons, and PWAs from one 1024px image."
canonical_url: "https://bestlatestapps.com/blog/how-to-generate-an-ios-appicon-set-and-android-icons-from-one/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-generate-an-ios-appicon-set-and-android-icons-from-one/).

{% raw %}

Every platform wants icons differently. Xcode wants an asset catalogue, iOS 18 added dark and tinted variants, macOS has its own grid, Android needs adaptive layers plus a monochrome one for themed icons, and PWAs want maskable versions. Getting any of it wrong means a rejected build or an icon with a white box around it. [App icon generator](https://bestlatestapps.com/tools/app-icon-generator/) takes one square image and outputs ready-to-drop folders for all of them, with a live preview of each platform's mask.

## Steps

1. Bring a 1024×1024 (or larger) square PNG with square, not rounded, corners. No artwork yet? The built-in **Icon maker** combines a gradient background, a glyph and a few letters (four at most).
2. Tick the platforms you ship: iOS/iPadOS, macOS, watchOS, visionOS, Android, web.
3. Look over the live previews, which show how each platform crops the artwork: Apple's rounded square, Android's circle and squircle shapes, the single-colour themed version, and the safe area a maskable web icon must respect.
4. Download the ZIP.

## What you get

- **iOS and iPadOS:** a 1024 icon with no alpha channel, plus the two iOS 18 variants: dark (artwork over transparency) and tinted (greyscale). (Or the 18 legacy sizes if you need them.) Since Xcode 14, one 1024 image is enough; Xcode generates the smaller sizes.
- **macOS:** 16 to 512 pt at 1x and 2x, laid out on Apple's Big Sur template (an 824 px rounded plate with a gentle shadow), or edge to edge if you switch the plate off.
- **watchOS:** one 1024 image, which the watch crops to a circle itself.
- **visionOS:** a layered image set.
- **Android:** every launcher density (mdpi up to xxxhdpi), separate foreground and background layers for adaptive icons, a single-colour layer so themed icons work on Android 13+, the `mipmap-anydpi-v26` XML that ties them together, and the 512 px listing icon for Google Play.
- **Web:** icons for a PWA manifest, including maskable ones.

## The rules that cause rejections

- **The main iOS 1024 icon must be opaque.** Upload one with an alpha channel and App Store Connect refuses it. The tool writes it as RGB and fills transparent areas with your chosen background colour. This is the number-one reason icons get rejected.
- **Don't pre-round the corners or add a drop shadow for iOS.** iOS crops the icon itself, and corners you rounded yourself leave a visible double edge.
- **Keep detail away from the edges.** Masks crop differently on each platform; the preview shows where.
- **Keep text short.** It's unreadable at 40 px. One or two letters at most.

## iOS 18 dark and tinted

For the dark variant, supply just the artwork over transparency and let iOS draw the dark background behind it. Tinted icons are grayscale; iOS applies the user's chosen tint. Keep the single-size option on and tick dark and tinted, and the ZIP adds both to Contents.json. If you've designed your own variants, upload them instead.

## A quick workflow

1. Export your logo at 1024×1024 from your design tool, no corners.
2. Check the Android circle preview. If the logo touches the edge, shrink it a bit in the source.
3. Download, drag the iOS AppIcon set into Xcode's asset catalogue and the Android `res` folders into your project.
4. Build and look at the home screen in light, dark and tinted modes.

## Privacy

The tool page says icons are drawn on a canvas in the tab and zipped there, and its code doesn't send your artwork anywhere. Letters typed in the icon maker aren't added to the page address. Like the rest of the site, the page shows Google ads.

## FAQ

**What size should an iOS icon be?**
1024×1024. Xcode generates the rest.

**Why was my icon rejected?**
Almost always transparency (an alpha channel) in the 1024 PNG.

**Does it make Android themed icons?**
Yes, with a monochrome layer and adaptive XML.

**Is my image uploaded?**
No, according to the page.

One image, every platform: try **[App icon generator](https://bestlatestapps.com/tools/app-icon-generator/)**.

*— Alok Kumar Singh*

{% endraw %}
