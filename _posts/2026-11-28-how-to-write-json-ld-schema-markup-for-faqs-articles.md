---
layout: post
title: "How to Write JSON-LD Schema Markup for FAQs, Articles and Breadcrumbs"
date: 2026-11-28 10:00:00 +0530
author: Alok Singh
description: "Generate JSON-LD structured data for FAQ, article and breadcrumb markup, where to put it, and what to check so search engines actually use it."
canonical_url: "https://bestlatestapps.com/blog/how-to-write-json-ld-schema-markup-for-faqs-articles/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-write-json-ld-schema-markup-for-faqs-articles/).

{% raw %}

Structured data is one of those SEO tasks that's easy to put off because writing JSON-LD by hand is fiddly and a single missing comma silently breaks it. [Schema markup generator](https://bestlatestapps.com/tools/schema-markup-generator/) builds valid schema.org JSON-LD for three common types from a simple form: FAQ, article and breadcrumbs.

## Steps

1. Choose **FAQ**, **Article** or **Breadcrumbs**.
2. Fill in the fields. Use **Add question** or **Add crumb** for more rows.
3. Copy the generated `<script type="application/ld+json">` block into your page.

Rows that are incomplete are left out automatically: a question with no answer, or a crumb with no URL.

## Which type for what

**FAQ** describes a list of questions and answers that appear on the page. Good for product pages, service pages and help articles that genuinely have an FAQ section.

**Article** describes a blog post or news piece: headline, author, dates, image. It helps search engines understand the content and who wrote it.

**Breadcrumbs** describe where the page sits in the site's hierarchy (Home → Tools → PDF → Merge PDF). Google can show this path in place of the raw URL in results.

Product, review, recipe and other types aren't covered here.

## Where it goes

In the `<head>` or the `<body>` of the page that shows the same content. JSON-LD is separate from the visible HTML, which is why Google recommends it over microdata: it's easier to add and harder to break when the design changes.

## What it won't do for you

Be realistic:

- It doesn't submit anything to Google.
- It doesn't guarantee a rich result. Google decides whether to show enhanced results, and FAQ rich results in particular are now shown only for a limited set of sites.
- Markup that doesn't match the visible page can be ignored, or in bad cases treated as spam. Every FAQ question in the markup should appear on the page, word for word or close to it.

## The check after publishing

Run the live URL through Google's Rich Results Test (and the Schema Markup Validator for general schema.org correctness). Then watch Search Console's Enhancements reports for errors over the following weeks.

## Example: a service page FAQ

A dental clinic page has five questions at the bottom: cost of a cleaning, insurance, parking, weekend hours, children. Paste each into the FAQ form, copy the script into the page template, publish, test. Even if Google never shows the expanded FAQ for this site, the markup costs nothing and keeps the page's structure explicit for any crawler that reads schema.org.

## Tips

- **Keep answers in plain text** in the markup; it's not the place for marketing fluff.
- **Breadcrumb URLs should be absolute** and match your real navigation.
- **Article dates:** update `dateModified` when you substantially change the content, not on every page load.
- **One FAQ block per page.** Don't repeat the same FAQ markup across many pages.
- **Validate after template changes.** A theme update can drop the script without anyone noticing.

## Why bother at all

Structured data doesn't directly boost rankings, but it removes guesswork. A search engine reading a page full of text has to infer which part is the headline, who the author is, and how the page fits into the site. JSON-LD states those facts plainly. When rich results do appear, they take up more space in results and tend to attract more clicks; when they don't, you've lost only a few minutes.

## Privacy

The page says the text stays in the tab and isn't uploaded. Google ads appear on the page, as across the site.

## FAQ

**Where does the script go?**
In the head or body of the page with the same content.

**Can it make product or review markup?**
No, only FAQ, article and breadcrumbs.

**Why did a row disappear?**
It was incomplete: a question without an answer, or a crumb without a URL.

**Will FAQ markup give me a rich result?**
Not necessarily; Google decides.

Add structured data in minutes with **[Schema markup generator](https://bestlatestapps.com/tools/schema-markup-generator/)**.

*— Alok Kumar Singh*

{% endraw %}
