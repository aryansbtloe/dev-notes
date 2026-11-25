---
layout: post
title: "How to Compress a Video for WhatsApp or Email, in Your Browser"
date: 2026-11-25 10:00:00 +0530
author: Alok Singh
description: "Shrink a video to fit WhatsApp's 16 MB or a 25 MB email limit in your browser, how target sizes work, and how to keep the result from looking blocky."
canonical_url: "https://bestlatestapps.com/blog/how-to-compress-a-video-for-whatsapp-or-email-in-your-browser/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-compress-a-video-for-whatsapp-or-email-in-your-browser/).

{% raw %}

Your kid's two-minute school performance, filmed in 4K on a phone, is 600 MB. WhatsApp won't send it as a normal video, email laughs at it, and the school's upload form caps at 25 MB. [Video compressor](https://bestlatestapps.com/tools/video-compressor/) re-encodes the clip to a size you choose and gives you an MP4 that will actually go through.

## Two ways to tell it what you want

**A target size.** Pick 16 MB (a sensible WhatsApp size), 25 MB (common email limit), or type your own. The page works out the video bitrate from the clip's length, leaves room for the audio and the file container, and drops the resolution if the picture would otherwise turn into blocks.

**A quality level.** High, balanced and small use a fixed amount of data per pixel instead of aiming at a size. Use these when you care about how it looks more than how big it is.

You can also set the resolution and frame rate yourself.

## Steps

1. Open the video.
2. Choose a target size or a quality, plus resolution and frame rate if you want control.
3. Press **Compress** and download the MP4.

## The physics of tight targets

A file size is just bitrate multiplied by duration. A 30-second clip at 16 MB gets plenty of data per second; a 10-minute clip at 16 MB gets very little, and video at a very low bitrate stops looking like video. The page keeps a minimum bitrate and warns you when the target is tight for the clip's length.

When that happens, you have three honest options:

- **Trim the clip.** Do you need all ten minutes? Cutting to the best two minutes helps more than any setting.
- **Lower the resolution.** 720p looks fine on a phone screen and needs far less data than 1080p or 4K.
- **Lower the frame rate.** 60 fps footage dropped to 30 fps halves the frames to encode.

## What you get

The video is H.264 when the browser can encode it, with AAC or Opus audio, in an MP4 that plays almost everywhere. Sound stays, at a bitrate you pick (96 or 128 kbps is typical). A silent video stays silent.

If the original was already under your target, the page tells you. Compressing still re-encodes it, but the bitrate stays close to the original, so the file rarely grows.

## Browser support

It uses each browser's built-in video encoder. Chrome, Edge and Safari work on computers and phones; Firefox works on computers. Firefox for Android has no video encoder available to the page, so use Chrome there.

## A worked example

That school performance: trim it to the 2 minutes your child is actually on stage, set 1280-pixel (720p) resolution, 30 fps, and a 25 MB target. Two minutes at roughly 1.6 Mbps looks perfectly good on a phone. Then send it to the family group or upload it to the form.

## Tips

- A computer encodes much faster than a phone, especially for long clips.
- Keep the phone plugged in and the screen on during a long compress on mobile.
- Keep the original. Compression is one-way; you can't get detail back from the small file.

## Privacy

According to the page, everything runs in the page and the MP4 is saved on your device when you press Download. The page shows Google ads, like the rest of the site.

## FAQ

**What size should I pick for WhatsApp?**
16 MB is the page's WhatsApp preset.

**What if the clip is too long for the target?**
It'll warn you. Trim, lower resolution, or accept a bigger file.

**Does the sound stay?**
Yes, at the bitrate you choose.

**Which browsers work?**
Chrome, Edge, Safari, and Firefox on desktop. Not Firefox on Android.

Got a video that won't send? Shrink it with **[Video compressor](https://bestlatestapps.com/tools/video-compressor/)**.

*— Alok Kumar Singh*

{% endraw %}
