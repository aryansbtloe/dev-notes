---
layout: post
title: "App Store Character Limits: How to Count Your Name, Subtitle and Keywords"
date: 2026-11-08 10:00:00 +0530
author: Alok Singh
description: "Count App Store and Google Play listing text against every limit, measure the keyword field in bytes, and find wasted keyword space before you submit."
canonical_url: "https://bestlatestapps.com/blog/app-store-character-limits-how-to-count-your-name-subtitle/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/app-store-character-limits-how-to-count-your-name-subtitle/).

{% raw %}
App Store Connect will tell you when a field is too long, but only after you paste, and it says nothing about wasted space. The 100-character keyword field is actually 100 **bytes**, which is a nasty surprise when you localise into Japanese or Russian. [App Store character counter](https://bestlatestapps.com/tools/app-store-character-counter/) counts every App Store and Google Play field live and flags keyword waste.

## The limits at a glance

**App Store:**

- App name: 2 to 30 characters
- Subtitle: 30
- Keywords: 100 bytes
- Promotional text: 170
- Description: 4,000
- What's New: 4,000

**Google Play:** app name 30, short description 80, full description 4,000. Google counts full-width and half-width characters the same.

Name, subtitle, keywords and description only change with a new version. **Promotional text** is the one field you can edit any time without submitting, so use it for seasonal messages and launches.

## Why bytes matter

The keyword field is measured in UTF-8 bytes. Basic Latin letters take one byte each; accented letters take two; Japanese, Chinese, Korean and Cyrillic characters take two or three. So a Russian keyword list fills up at roughly half the characters you'd expect. The counter shows bytes for that field specifically.

## Keyword checks

Based on Apple's guidance that your app is already searchable by its name and company name (and that you shouldn't use other apps' or companies' names), plus the way the field is parsed, the tool flags:

- **spaces after commas.** Keywords are comma-separated, so each space is a wasted byte;
- **repeated keywords;**
- **words already in your app name;**
- **singular/plural pairs** that likely don't need both;
- **very short keywords** that rarely pull their weight.

## Worked example

Imagine an app called "Habit Garden: Daily Tracker" with keywords `habit, garden, daily, tracker, habits, routine, goals`. The checks flag six spaces after commas, four words that are already in the name, and the habit/habits pair. Rewritten as `routine,goals,streak,reminder,planner,selfcare,mindful,journal`, the field carries eight new words with bytes to spare.

No counter can promise a ranking; this just stops you wasting the space you have.

## Previews

There's an approximate search result preview showing where long names and subtitles get cut with an ellipsis. The exact cut point depends on device width, so treat it as a guide, and keep the important words at the start of the subtitle.

## Drafts and export

The tool's code doesn't send your text anywhere. If you tick **Save drafts in this browser**, fields are kept in local storage so they survive a reload; **Clear all** wipes them. **Copy all fields** and **Copy as JSON** put a labelled block of every field on the clipboard, which is handy for pasting into App Store Connect, Play Console or a translator's spreadsheet.

## Tips

- Write the subtitle as a benefit, not a feature list: "Build habits that stick" beats "Tracker, reminders, stats".
- Draft localised keyword fields with a native speaker, then check them here for bytes.
- Put your strongest phrase in the subtitle, which is visible to users, and use the hidden keyword field for supporting terms.
- Revisit keywords every few releases based on what search terms actually convert.
- For icons to go with the listing, see [App icon generator](https://bestlatestapps.com/tools/app-icon-generator/).

## FAQ

**Do spaces count in the keyword field?**
Yes, every byte counts. Separate keywords with commas only.

**Should I repeat title words in keywords?**
Usually not, per Apple's guidance; spend the bytes on new words.

**Can I change promotional text without a release?**
Yes, it's the only field that allows it.

**What are the Google Play limits?**
Name 30, short description 80, full description 4,000.

Before your next submission, paste your listing into **[App Store character counter](https://bestlatestapps.com/tools/app-store-character-counter/)**.

*— Alok Kumar Singh*
{% endraw %}
