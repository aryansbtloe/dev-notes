---
layout: post
title: "How to Compare Two Texts and See the Differences"
date: 2026-11-26 10:00:00 +0530
author: Alok Singh
description: "Compare two versions of a text and see added, removed and changed lines with word-level highlights, using the same diff method as Git, and export a patch."
canonical_url: "https://bestlatestapps.com/blog/how-to-compare-two-texts-and-see-the-differences/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-compare-two-texts-and-see-the-differences/).

{% raw %}

A contract comes back from the other side with "a few minor edits". Your manager rewrote two paragraphs of your report "slightly". A config file worked yesterday and doesn't today. In each case the question is the same: what exactly changed? [Compare two texts](https://bestlatestapps.com/tools/text-compare/) lines up two texts and marks every added, removed and edited line, down to the changed words.

## Steps

1. Paste or open the older text on the left.
2. Paste or open the newer text on the right.
3. Read the marked lines, switch views, or copy the result as a patch.

## How it works

The page finds the smallest set of added and removed lines that turns the first text into the second, using the Myers diff method, the same approach Git uses. That's why inserting a line at the top shows as a single added line instead of making everything below look different.

Within edited lines, the changed words are highlighted on both sides, so a contract where "30 days" became "15 days" jumps out instead of hiding in a long paragraph.

Long runs of unchanged lines are folded into a short note, keeping 3 lines of context around each change. Tick **Show every line** to see everything.

## Options worth knowing

**Ignore spaces.** When ticked, extra, leading and trailing spaces and tabs don't count as differences. Without it, a trailing space counts as a change, which is right for code and wrong for prose that's been reformatted.

**Patch output.** Copy the result as a unified diff (the `git diff` format) with 3 lines of context. Share it, or apply it to the original with `git apply` or `patch`.

## What it doesn't do

- **Moved blocks** show as removed in one place and added in another; moves aren't labelled.
- **Very different texts:** up to 10,000 lines per side are supported, but when two long texts differ almost everywhere, the middle is shown as one changed block, and the page tells you.
- **PDF and Word files:** paste their text instead, or use [Compare PDF](https://bestlatestapps.com/tools/compare-pdf/) for two PDFs. The page opens plain-text files like .txt, .md, .csv, .json and code files.

## Real uses

- **Contract and policy reviews.** Paste the previous and new versions and check every highlighted change, not just the ones the other party mentioned.
- **Editing.** See exactly what an editor changed in your article.
- **Config and code.** Compare a working and broken config file, or two API responses (format JSON first with [JSON formatter](https://bestlatestapps.com/tools/json-formatter/) so the lines align).
- **Translations and subtitles.** Spot lines that went missing.
- **Data exports.** Compare two CSV exports to see which rows changed.

## Example: a rental agreement revision

Landlord sends a "same as last year" agreement. Paste last year's on the left, this year's on the right, tick Ignore spaces since the formatting changed. Three highlighted changes appear: rent up 8%, notice period down from two months to one, and a new clause about painting charges. Without the diff, the last two would probably have slipped past.

## Privacy

The page says both texts and any files you open stay in the tab and aren't uploaded, which is what you want for contracts. Google ads appear on the page, as across the site.

## FAQ

**Does it ignore extra spaces?**
Only when you tick Ignore spaces.

**Does it track moved paragraphs?**
They show as removed and added; moves aren't labelled.

**How big can the texts be?**
Up to 10,000 lines per side.

**What's the patch file?**
A unified diff you can apply with git apply or patch.

Before you sign or merge, check what changed with **[Compare two texts](https://bestlatestapps.com/tools/text-compare/)**.

*— Alok Kumar Singh*

{% endraw %}
