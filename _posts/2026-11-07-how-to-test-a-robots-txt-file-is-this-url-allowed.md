---
layout: post
title: "How to Test a robots.txt File: Is This URL Allowed?"
date: 2026-11-07 10:00:00 +0530
author: Alok Singh
description: "Test whether a robots.txt file allows or blocks a URL for Googlebot or any crawler, see which rule matched, and understand longest-match precedence and wildcards."
canonical_url: "https://bestlatestapps.com/blog/how-to-test-a-robots-txt-file-is-this-url-allowed/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-test-a-robots-txt-file-is-this-url-allowed/).

{% raw %}
A single misplaced `Disallow: /` has de-indexed entire sites. More commonly, a rule meant to block `/admin` also blocks `/administration-guide`, the most useful page on the site. [Robots.txt tester](https://bestlatestapps.com/tools/robots-txt-tester/) tells you, for a given crawler and URL, whether it's allowed or blocked and exactly which line decided it.

## Steps

1. Paste the robots.txt content.
2. Set the crawler name. Googlebot is the default.
3. Enter a full URL or a path like `/private`.

You get "allowed" or "blocked" plus the rule that matched.

The page doesn't fetch the live file. You paste what you want tested, which is actually useful: you can test a draft *before* deploying it.

## How the winner is chosen

This is where intuition fails, so it's worth knowing the logic:

1. **Pick the group.** The tool chooses the `User-agent` group whose name appears in the crawler name, preferring the longest matching name. If none match, the `*` group applies.
2. **Find matching rules.** Empty `Disallow:` and empty `Allow:` lines don't match anything.
3. **Longest pattern wins.** Among matching rules, the longer path pattern takes precedence.
4. **Ties go to Allow.** Same length? Allow wins.

So with:

```
User-agent: *
Disallow: /private
Allow: /private/ok
```

`/private/ok` is allowed (longer Allow beats shorter Disallow), and `/private/secret` is blocked.

## Wildcards

- `*` matches any sequence of characters.
- `$` at the end anchors to the end of the path.

`Disallow: /*.pdf$` blocks URLs ending in .pdf, but not `/file.pdf?download=1`, because that doesn't end in .pdf.

Lines after `#` are comments and ignored.

## Common mistakes

- **`Disallow: /admin`** also blocks `/admin-tips` and `/administrator`. Use `/admin/` with the trailing slash if you mean the directory.
- **Blocking CSS and JS folders.** Google needs them to render pages; blocking them can hurt how your pages are understood.
- **Using robots.txt to hide pages from search.** Blocked pages can still be indexed (without content) if others link to them. For "don't show in search", use a `noindex` meta tag and leave the page crawlable.
- **A staging robots.txt shipped to production** with `Disallow: /`. Test your production file here after every deploy.
- **Case sensitivity.** Paths are case-sensitive: `/Private` and `/private` are different.

## Example: launching a new section

You're adding `/blog/drafts/` that shouldn't be crawled, while `/blog/` must stay open. Add `Disallow: /blog/drafts/`, then test `/blog/my-post/` (should be allowed), `/blog/drafts/x/` (blocked) and `/blog/drafts-and-ideas/` (allowed, thanks to the trailing slash). Three checks, and you're confident before deploying.

## Limits

It applies the rules in the box. It doesn't ask Google, and the live index can differ, for instance if Google cached an older robots.txt. Use Search Console's tools for what Google has actually seen.

## A pre-deploy checklist

Keep a short list of URLs that must always be crawlable (home page, key category pages, a top product, the sitemap) and a few that must always be blocked (admin, cart, internal search results). Before shipping any robots.txt change, paste the new file and run each URL through the tester. It takes two minutes and catches the accidental block before Google does. Rerun the same list after a CMS or hosting migration, when robots.txt files often get silently replaced.

## Privacy

The tool page says the file and URL stay in the tab and aren't uploaded. Google ads run on the page, as across the site.

## FAQ

**Why is /private/ok allowed when /private is blocked?**
The longer Allow pattern wins.

**Does it check with Google?**
No, it applies the rules you pasted.

**Does `Disallow: /` block everything?**
Yes, for that group, unless a longer Allow matches.

**Which crawler should I test?**
The one you care about, usually Googlebot.

Before the next robots.txt change goes live, test it with **[Robots.txt tester](https://bestlatestapps.com/tools/robots-txt-tester/)**.

*— Alok Kumar Singh*
{% endraw %}
