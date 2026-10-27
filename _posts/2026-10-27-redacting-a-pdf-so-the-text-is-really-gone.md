---
layout: post
title: "Redacting a PDF So the Text Is Really Gone: Rebuilding Pages with pdf.js and pdf-lib, Then Checking the Result"
date: 2026-10-27 10:00:00 +0530
author: Alok Singh
description: "Why a black rectangle isn't a redaction, how my browser tool finds words in PDF coordinates, rasterises only the pages that need it, and refuses to save until the hidden text can't be found"
canonical_url: "https://bestlatestapps.com/blog/redacting-a-pdf-so-the-text-is-really-gone/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/redacting-a-pdf-so-the-text-is-really-gone/).

{% raw %}
### Why a black rectangle isn't a redaction, how my browser tool finds words in PDF coordinates, rasterises only the pages that need it, and refuses to save until the hidden text can't be found

There's a familiar kind of news story: a document is published with black bars over the sensitive parts, someone selects the text under the bars, pastes it, and there it all is. The bars were shapes drawn on top. The words were still in the file.

That failure is the whole design brief for the [Redact PDF](https://bestlatestapps.com/tools/redact-pdf/) tool on my site. A redaction tool has one job, and "looks redacted" isn't it. This article explains how the tool makes the text actually go away, using its real source, `pdf-redact.js`, and the two open-source libraries it builds on: **pdf.js** (Mozilla's PDF renderer, the one Firefox uses) to read and draw pages, and **pdf-lib** to write the new file. Snippets are taken or trimmed from the tool's code; where I generalise, I say so.

------------------------------------------------------------------------

## 1. Why drawing a box isn't redacting

A PDF page is a program: a content stream of drawing operators. Text is drawn with text operators that reference a font and carry the characters. When an editor "adds a black box", it usually appends a rectangle fill to that stream, or adds an annotation. The text operators are untouched. A viewer paints the box over the glyphs, but text extraction, search, copy and paste and screen readers read the operators, not the pixels.

Real redaction has to remove the text operators, along with anything else that could carry the content: annotations, form field values, alternative text. There are two broad ways to do that:

1.  **Edit the content stream.** Find the text operators that fall under each box and remove or rewrite them. This keeps the rest of the page as live text, but it's hard to get right: fonts with custom encodings, text drawn as paths, overlapping objects, form XObjects nested inside each other.
2.  **Rebuild the page as a picture.** Render the page to pixels, paint the boxes into those pixels, and replace the page with an image. Nothing from the original page's content survives except what's visible.

The tool takes the second route, deliberately. It costs selectable text on the pages you redact, and it's very hard to get wrong. For a free tool that people will use on bank statements and ID scans, I'll take "hard to get wrong" every time.

------------------------------------------------------------------------

## 2. Only rebuild what needs rebuilding

Rasterising a 40-page contract because one page has an account number would be a poor trade. So the tool tracks which pages have at least one box and rebuilds only those:

``` javascript
/** Sorted unique page numbers (1-based) that have at least one box. Pure. */
function affected(boxes) {
  var seen = {};
  boxes.forEach(function (b) { seen[b.page] = 1; });
  return Object.keys(seen).map(Number).sort(function (a, b) { return a - b; });
}
```

Every other page is copied unchanged with pdf-lib's `copyPages`, so its text stays selectable and searchable. Before you press the button, the page tells you exactly what will happen, generated from the same function:

> Before you save: pages 2, 5 and 7 will be rebuilt as pictures with the black boxes burned in. Text on those pages can no longer be selected, searched or copied, and links and form fields there are dropped. Other pages are copied unchanged.

That message isn't decoration. It's the honest description of the trade-off, shown at the moment someone can still change their mind.

------------------------------------------------------------------------

## 3. Drawing boxes in PDF space, not screen space

The preview is a `<canvas>` rendered by pdf.js, with a transparent layer on top that captures pointer events. The key decision is that every box is stored in **PDF units at scale 1**, the page's own coordinate system, never in screen pixels:

``` javascript
function toPage(ev) {
  var r = layer.getBoundingClientRect();
  return { x: (ev.clientX - r.left) / r.width * view1.width, y: (ev.clientY - r.top) / r.height * view1.height };
}
```

`view1` is `page.getViewport({ scale: 1 })`. Because the pointer position is converted to a fraction of the layer's size and then multiplied by the page's size in points, the stored box doesn't care how big the preview is, how the browser is zoomed, or what the device pixel ratio is. Drawn boxes are displayed with percentage positions, so they stay glued to the page when the window resizes.

A few small details make drawing feel right:

- `setPointerCapture` keeps the drag going if the pointer leaves the layer.
- Boxes smaller than 3 by 3 points are discarded, so a stray click doesn't create an invisible redaction.
- The preview canvas uses a device pixel ratio capped at 2, which keeps it sharp on most screens without allocating huge canvases on dense displays.
- Each redaction is listed below the page with a Remove button, labelled with its page and either the search term or "box you drew".

------------------------------------------------------------------------

## 4. Finding words and placing boxes over them

Drawing boxes by hand is fine for a signature or a photo. For "every occurrence of this account number", the tool searches the text. pdf.js's `getTextContent()` returns text items: runs of characters with a transform matrix, a width and a height. The pure `locate()` function turns matches inside those items into boxes:

``` javascript
function locate(items, term, viewTransform, scale, mul) {
  var out = [], q = String(term || "").toLowerCase();
  if (!q) return out;
  items.forEach(function (it) {
    var s = String(it.str || ""), low = s.toLowerCase();
    if (!s || low.indexOf(q) < 0) return;
    var tx = mul(viewTransform, it.transform);
    var h = Math.hypot(tx[2], tx[3]) || Math.abs(it.height || 10);
    var w = (it.width || 0) * scale;
    var per = s.length ? w / s.length : 0;
    for (var at = low.indexOf(q); at >= 0; at = low.indexOf(q, at + q.length)) {
      var pad = h * 0.18;
      out.push({ x: tx[4] + per * at - pad, y: tx[5] - h - pad, w: per * q.length + 2 * pad, h: h * 1.25 + 2 * pad });
    }
  });
  return out;
}
```

There's a lot packed in here:

- **Coordinate systems.** PDF space has its origin at the bottom left with y going up. The viewport's transform flips it to top-left, y down. Multiplying the viewport transform by the item's own transform (the tool passes in `pdfjsLib.Util.transform` as `mul`) gives the item's baseline position in the same space the boxes use.
- **Font size from the matrix.** The vertical scale of the combined matrix, `hypot(tx[2], tx[3])`, is the rendered font size, which also works for rotated text.
- **Character positions are estimated.** A text item gives its total width, not per-glyph positions. The code assumes an even width per character, `width / length`. That's exact for monospaced digits and close for most proportional text, and the padding (18% of the font height on every side, and a height of 1.25 times the font size) absorbs the difference and covers ascenders and descenders.
- **Every occurrence.** The loop restarts the search after each match, case-insensitively, so "Smith" finds "SMITH" and every repeat in the same run.

Making `locate` pure, with the matrix multiply passed in, means it can be tested without a browser or a PDF. I did exactly that while writing this: a 12-point item at (72, 700) on a US Letter page, searching for "1234" inside "Account: 1234 5678", gives a box starting about 130 points from the left and 78 from the top, which is where those digits sit.

### When the word can't be placed

Two situations defeat `locate`. A word can be split across text items (some PDF producers emit one item per word or even per glyph), or the match can sit in an annotation, such as a comment or a form field value, rather than in the page text. The tool checks a joined string of all items *plus* annotation text, and if the term is there but no box could be placed, it marks the whole page for rebuilding:

``` javascript
var low = st.joined.toLowerCase().replace(/\s+/g, " ");
if (low.indexOf(term.toLowerCase()) >= 0 && !hits.length) {
  // split across text pieces, or only in a comment or form value: cover the whole page so it is still removed
  boxes.push({ page: n, x: 0, y: 0, w: 0, h: 0, term: term, whole: true });
  unplaced.push(n); pagesHit[n] = 1;
}
```

It's worth being precise about what that does. Rebuilding removes the word from the file as *text*: it can no longer be searched, copied or extracted. But the rebuilt picture shows whatever the page showed, so if the word was visible, it's still visible as pixels until you draw a box over it. That's why the status message names those pages and asks you to draw a box there. If a word can't be located automatically, a person has to look.

And when the term isn't in the text at all, the message says so and points at the likely reason: it's in a scan or an image, where there are no letters to find.

------------------------------------------------------------------------

## 5. Rebuilding the redacted pages

When you press "Redact and download", the tool creates a new, empty document and walks the pages in order:

``` javascript
var out = await PDFLib.PDFDocument.create();
var keep = [];
for (var i = 1; i <= doc.numPages; i++) if (pages.indexOf(i) < 0) keep.push(i - 1);
var copied = src && keep.length ? await out.copyPages(src, keep) : [];
for (var n = 1; n <= doc.numPages; n++) {
  if (pages.indexOf(n) < 0) { out.addPage(copied[ci++]); continue; }
  var page = await doc.getPage(n), v1 = page.getViewport({ scale: 1 });
  var sc = fitScale(v1.width, v1.height, dpi / 72), v = page.getViewport({ scale: sc });
  var c = document.createElement("canvas"); c.width = Math.floor(v.width); c.height = Math.floor(v.height);
  var cx = c.getContext("2d");
  cx.fillStyle = "#fff"; cx.fillRect(0, 0, c.width, c.height);
  await page.render({ canvasContext: cx, viewport: v }).promise;
  cx.fillStyle = "#000";
  boxes.forEach(function (b) { if (b.page === n && !b.whole) cx.fillRect(Math.floor(b.x * sc), Math.floor(b.y * sc), Math.ceil(b.w * sc) + 1, Math.ceil(b.h * sc) + 1); });
  var blob = await new Promise(function (r) { c.toBlob(r, "image/jpeg", 0.92); });
  c.width = c.height = 0;
  var img = await out.embedJpg(new Uint8Array(await blob.arrayBuffer()));
  out.addPage([v1.width, v1.height]).drawImage(img, { x: 0, y: 0, width: v1.width, height: v1.height });
}
```

The choices, one by one:

- **Resolution is the user's.** 200 DPI by default, 150 for a smaller file, 300 for print. The scale is `dpi / 72`, since PDF units are points.
- **A pixel budget.** `fitScale` caps the canvas at about 16 megapixels: `Math.min(want, Math.sqrt(16e6 / (w * h)))`. An A4 page at 300 DPI is under 9 megapixels, so normal pages get the resolution you asked for, and an oversized drawing sheet is scaled down instead of failing to allocate.
- **White first.** The canvas is filled white before rendering. Many PDFs have no page background, and a transparent canvas would turn black when encoded as JPEG.
- **Floor the origin, ceil the size, add one.** Boxes are expanded outward to whole pixels plus a pixel of margin, so anti-aliasing can't leave a sliver of a glyph at the edge.
- **JPEG at quality 0.92.** Scanned-looking pages compress far better as JPEG than PNG, and 0.92 keeps text crisp. The canvas is released (`c.width = c.height = 0`) straight after encoding, which helps memory on long documents.
- **Same page size.** The image is drawn to fill a page of exactly the original size in points, so the document still prints and paginates the same way.

Because the output is a fresh document, document-level information from the original, such as the title and author metadata, isn't carried over. The page notes that too.

------------------------------------------------------------------------

## 6. Check the result before offering it

This is the part I'd insist on in any redaction tool, and the part most tools skip. After writing the file, the tool opens the *output* with pdf.js and looks for what should be gone:

``` javascript
var check = await pdfjsLib.getDocument({ data: result.slice() }).promise, left = [];
for (var k = 1; k <= check.numPages; k++) {
  var st = await pageStrings(await check.getPage(k));
  if (pages.indexOf(k) >= 0 && st.items.some(function (it) { return String(it.str).trim(); })) left.push("text on page " + k);
  var low = st.joined.toLowerCase();
  terms.forEach(function (t) { if (low.indexOf(t.toLowerCase()) >= 0) left.push("“" + t + "” on page " + k); });
}
if (left.length) throw new Error("Not saved: the check still found " + left.join(", ") + ". Find the word again, or draw a box on that page, then save.");
```

Two independent checks run on the new file:

1.  **Rebuilt pages contain no text at all.** They should be a single image. Any extractable text means something went wrong.
2.  **Every searched term is absent from every page**, including annotation strings, across the whole document, not only the rebuilt pages.

If either check fails, no download is offered, and the message says what was found and where. When it passes, the status line says what was checked, and still ends with "Open it and look before you share it." A program can confirm that text is absent. Only a person can confirm that nothing sensitive is visible.

Verifying output with a *different code path* from the one that produced it is a pattern I'd recommend well beyond PDFs. Here, pdf-lib wrote the file and pdf.js reads it back, so a bug in how the file was written shows up as a failed check rather than a leaked document.

------------------------------------------------------------------------

## 7. Trade-offs and limits, stated plainly

Every approach to redaction has costs. These are this one's:

- **Redacted pages lose live text.** They can't be searched, selected or read by a screen reader, and links and form fields on them are gone. If you need those pages accessible, you'd want an OCR pass afterwards; the site's [OCR PDF](https://bestlatestapps.com/tools/ocr-pdf/) tool can add a text layer, which of course won't include what's under the boxes.
- **Find only finds letters.** A scanned page is a picture, so Find has nothing to match. Draw boxes on scans.
- **Manual boxes are checked for "no text on the page", not for meaning.** The tool can't know that the thing you boxed was the thing that needed boxing.
- **Unchanged pages are copied as they are.** If a page with no redaction contains something sensitive that you didn't search for, it's still there. Search for every variant: with and without spaces, formatted and unformatted numbers.
- **Password-protected files** have to be opened first. The tool detects pdf.js's `PasswordException` and points to [Unlock PDF](https://bestlatestapps.com/tools/unlock-pdf/), which only works with a password you know.
- **Files that pdf-lib can't copy page by page** stop with a message suggesting Repair PDF, rather than producing a partial document.

On privacy: the PDF is read into memory in the tab, pdf.js renders it there, and pdf-lib writes the new file there. The tool's code doesn't upload the document or the words you search for, and the search field has browser autocomplete turned off, so the term isn't offered back to the next person who uses the same browser. For a tool whose entire purpose is protecting sensitive text, sending the file to a server first would defeat the point.

------------------------------------------------------------------------

## 8. A redaction checklist

1.  **Search for every form of the sensitive value**: names with and without initials, numbers with and without separators.
2.  **Draw boxes over anything in scans, images or signatures**, where Find can't see.
3.  **Read the page list in the warning** and make sure losing live text on those pages is acceptable.
4.  **Draw a box wherever the tool says a match couldn't be placed.**
5.  **Pick 200 DPI** unless the file is for print.
6.  **Let the check run**, and treat "Not saved" as the tool doing its job.
7.  **Open the result and confirm it with your eyes**, then confirm with Ctrl+F or [PDF to text](https://bestlatestapps.com/tools/pdf-to-text/).
8.  **Share the redacted file, never the original**, and keep the original somewhere private.

------------------------------------------------------------------------

## Wrapping up

Redaction fails when a tool optimises for how a page looks rather than what a file contains. Rebuilding only the affected pages as pictures, placing boxes in PDF coordinates, and refusing to save until a separate reader confirms the text is gone turns "looks redacted" into "is redacted". The cost, losing live text on those pages, is shown to you before you commit.

Try it, or browse the related PDF tools:

- **Redact PDF:** https://bestlatestapps.com/tools/redact-pdf/
- **PDF to text:** https://bestlatestapps.com/tools/pdf-to-text/
- **Unlock PDF:** https://bestlatestapps.com/tools/unlock-pdf/
- **How I built a document scanner:** https://bestlatestapps.com/blog/a-document-scanner-in-an-afternoon/
- **All of my free web tools:** https://bestlatestapps.com/tools/
- **My portfolio:** https://bestlatestapps.com/alok/

If you've ever received a "redacted" document where the hidden text was still selectable, you already know why the tool refuses to save until the check passes.

*— Alok Kumar Singh*
{% endraw %}
