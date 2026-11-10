---
layout: post
title: "How to Check a GSTIN or PAN Number: Format, State Code and Check Digit"
date: 2026-11-10 10:00:00 +0530
author: Alok Singh
description: "Check whether a GSTIN or PAN is correctly formed: state code, embedded PAN, holder type and the GSTIN check character, and what a valid result does and doesn't prove."
canonical_url: "https://bestlatestapps.com/blog/how-to-check-a-gstin-or-pan-number-format-state-code-and-check/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/how-to-check-a-gstin-or-pan-number-format-state-code-and-check/).

{% raw %}

A supplier's invoice arrives with a GSTIN, and you need to claim input tax credit on it. Or you're entering 200 vendor GSTINs into accounting software and want to catch typos before they cause mismatches in GSTR-2B. [GSTIN and PAN checker](https://bestlatestapps.com/tools/gstin-validator/) checks the structure of a GSTIN or PAN and tells you what each part means.

## Steps

1. Paste the GSTIN or PAN.
2. Read the state, the PAN and the holder type.
3. For a GSTIN, compare the check character with the one the first 14 characters produce.

## Anatomy of a GSTIN

A GSTIN has 15 characters:

1. **Characters 1–2:** state code (27 is Maharashtra, 29 Karnataka, 07 Delhi, and so on).
2. **Characters 3–12:** the holder's PAN.
3. **Character 13:** entity number, meaning which registration this is for that PAN in that state. 1 is the first; later ones use 2, 3 and then letters.
4. **Character 14:** a letter, usually Z.
5. **Character 15:** a check character computed from the first 14 with a base-36 weighted sum.

The page's sample, `27ABCPE1234F1ZB`, decodes as: Maharashtra, PAN ABCPE1234F, an individual, first registration, check character B.

## Anatomy of a PAN

Five letters, four digits, one letter. The **fourth letter** is the holder type published by the Income Tax Department: P for an individual, C for a company, F for a firm, H for a Hindu Undivided Family, T for a trust, and others. A PAN with C in that position on an invoice from "Mr Sharma" is worth a second look.

A PAN alone has no extra check digit that the page can verify. The GSTIN's final character is what adds that.

## Old and new state codes

A couple of codes cause confusion: 25 was Daman and Diu's earlier code, and 26 is now the merged union territory. Andhra Pradesh moved from 28 to 37 after bifurcation. The checker knows these.

## What a "valid" result means

A matching check character means the GSTIN was typed correctly: the last character fits the first 14. It catches most typos and transposed characters.

It does **not** mean the business is registered, active, or filing returns. Registration status lives on the GST portal, which has a public search by GSTIN. For any significant purchase where you'll claim credit, check there too.

## Practical uses

- **Accounts payable:** validate GSTINs as you onboard vendors, before the first invoice.
- **Invoice checks:** confirm the GSTIN on a bill matches the supplier's state (a Karnataka supplier with a 27- GSTIN is a red flag, unless they have a Maharashtra registration).
- **Data cleanup:** run through a vendor master list and fix malformed entries.
- **Your own invoices:** make sure your GSTIN is printed correctly on your template. A typo there causes your customers headaches.

## Tips

- Copy GSTINs from the source rather than retyping; O and 0, I and 1 are easy to confuse.
- Store GSTINs uppercase without spaces.
- For creating GST-compliant invoices, see [Invoice generator](https://bestlatestapps.com/tools/invoice-generator/).

## Privacy

The page says everything runs in the page and stays on your device. Google ads appear on the page, as on the rest of the site.

## FAQ

**What does a matching check character tell me?**
That the GSTIN is correctly formed. Status is on the GST portal.

**What's the entity number?**
The 13th character: which registration this is for the PAN in that state.

**Why is there no check for PAN?**
A PAN has no separate check digit; the format and holder type are what can be checked.

**Which old state codes are handled?**
25/26 for Daman and Diu, 28/37 for Andhra Pradesh.

Catch GSTIN typos before they cost you credit: **[GSTIN and PAN checker](https://bestlatestapps.com/tools/gstin-validator/)**.

*— Alok Kumar Singh*

{% endraw %}
