---
layout: post
title: "Generating Swift Codable Types from JSON in the Browser: Number Literals, Optional Merging and the convertFromSnakeCase Rules"
date: 2026-10-27 11:00:00 +0530
author: Alok Singh
description: "Inside my JSON to Swift Codable converter: a parser that keeps 1.0 a Double, type inference that merges array items, naming that survives Swift keywords, and a decoder snippet that matches the\u2026"
canonical_url: "https://bestlatestapps.com/blog/generating-swift-codable-types-from-json-in-the-browser/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/generating-swift-codable-types-from-json-in-the-browser/).

{% raw %}
### Inside my JSON to Swift Codable converter: a parser that keeps 1.0 a Double, type inference that merges array items, naming that survives Swift keywords, and a decoder snippet that matches the dates it found

Every iOS project eventually gets a new endpoint, a JSON sample in a ticket, and twenty minutes of typing `let` properties and `CodingKeys` cases by hand. It's dull work, and it's also where real bugs come from: a field that's `null` in one response but not the sample, a price that arrives as `1.0`, a date with milliseconds that `.iso8601` refuses.

I built a [JSON to Swift Codable converter](https://bestlatestapps.com/tools/json-to-swift-codable/) for my site to do that typing. The interesting part isn't printing `struct`; it's the decisions in between. This article goes through them using the tool's actual source, `swift-codable-core.js`. The snippets are trimmed from it, and the generated Swift you'll see is real output: I ran the core in Node while writing this.

------------------------------------------------------------------------

## 1. Why `JSON.parse` is the wrong first step

The obvious design is `JSON.parse(text)`, then walk the result. It loses information a Swift generator needs:

- **Number literals.** `JSON.parse("1.0")` gives the JavaScript number `1`, indistinguishable from `JSON.parse("1")`. But an API that writes `1.0` is telling you the field is fractional, and if you type it `Int`, the first `1.5` breaks decoding.
- **Duplicate keys.** `JSON.parse` silently keeps the last value. Swift's `JSONDecoder` isn't guaranteed to agree, so a generator should at least warn.
- **Prototype-shaped keys.** If you collect keys in a plain object, a JSON key called `constructor` or `__proto__` can collide with JavaScript's own properties.

So the tool has its own small recursive-descent parser. It doesn't build JavaScript values; it builds a typed tree: `{t: "obj", keys, vals}`, `{t: "arr", items}`, `{t: "str", v}`, `{t: "int", raw}` and so on.

------------------------------------------------------------------------

## 2. A parser that keeps what the JSON actually said

Numbers are matched against the JSON grammar and classified by their spelling:

``` javascript
function num() {
  var m = /^-?(0|[1-9]\d*)(\.\d+)?([eE][+-]?\d+)?/.exec(s.slice(i, i + 400));
  if (!m) err("Bad number");
  i += m[0].length;
  var isInt = !m[2] && !m[3], big = isInt && Math.abs(Number(m[0])) > 9223372036854775807;
  return { t: isInt && !big ? "int" : "dbl", raw: m[0] };
}
```

A number with a fraction *or* an exponent is a Double, even `1.0` or `1e3`. An integer outside the 64-bit range also becomes Double, because Swift's `Int` can't hold it. One honest footnote: that comparison runs in floating point, where `9223372036854775807` rounds to 2<sup>63</sup>, so the cut-off is only exact to within the rounding step at that size. In practice IDs near 2<sup>63</sup> are rare, but if your API sends them, use `String`.

Objects keep their keys in order, store values in a map with no prototype, and record duplicates:

``` javascript
function dict() { return Object.create(null); }
function has(m, k) { return Object.prototype.hasOwnProperty.call(m, k); }
// inside obj():
var at = i, k = str();
if (has(vals, k)) dups.push({ key: k, line: s.slice(0, at).split("\n").length });
// ...
if (!has(vals, k)) keys.push(k);
vals[k] = v;
```

`Object.create(null)` plus own-property checks means `"constructor"` and `"__proto__"` are just keys. Feed the converter `{"__proto__": 1}` and you get a property named `proto` with `case proto = "__proto__"` in its CodingKeys.

Errors carry a line and column, computed from the current index: `"Expected ',' or '}' at line 4, column 12"`. When someone pastes a truncated log line, that's the difference between fixing it in five seconds and staring at it.

------------------------------------------------------------------------

## 3. Type inference is a merge, not a lookup

For a single value, inference is a `switch`: null, bool, int, double, string. Strings get two extra checks, both toggles in the UI:

``` javascript
// seconds are required: .iso8601 (ISO8601DateFormatter, internet date-time) rejects "2026-10-25T10:00Z"
var ISO_RE = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d+)?(Z|[+-]\d{2}:\d{2})$/;
var URL_RE = /^https?:\/\/[^\s]+$/i;
```

The comment explains why the date pattern insists on seconds. A string like `2026-10-25T10:00Z` looks like a date, but `JSONDecoder`'s `.iso8601` strategy rejects it, so typing it `Date` would produce code that fails at runtime. It stays a `String`.

The real work is in arrays. An API returns `posts: [...]`, and the elements don't all have the same keys. The tool infers each element and folds them together with `merge`:

``` javascript
function merge(a, b) {
  if (a.k === "null" && b.k === "null") return a;
  if (a.k === "null") return withOpt(b);
  if (b.k === "null") return withOpt(a);
  // ...
  if (a.k === b.k) { /* recurse into arrays and objects, OR together date "has fraction" */ }
  else if ((a.k === "int" && b.k === "dbl") || (a.k === "dbl" && b.k === "int")) r = { k: "dbl" };
  else if (/* date or url mixed with str, or date with url */) r = { k: "str" };
  else r = { k: "any" };
  return withOptIf(r, opt);
}
```

Read as rules:

- **`null` plus anything** makes that thing optional.
- **`Int` plus `Double`** is `Double`.
- **A date or URL plus a plain string** is a `String`; one non-date value means the field isn't reliably a date.
- **Anything else mixed** (a number and a string, say) is `any`, which becomes `JSONValue` in the output.

Objects merge field by field, and each merged object type counts how many objects it has seen (`count`) and how many times each key appeared (`seen`). A key with `seen[k] < count` was missing from at least one element, so it's optional:

``` javascript
function optional(t, k) { var f = t.fields[k]; return !!(f.opt || f.k === "null" || t.seen[k] < t.count); }
```

That's the rule I'd want from a colleague: optional when the sample proves it can be absent or null, required otherwise. The flip side is on the page too: the sample shapes the types. A key that's present in every element you pasted will be non-optional, so paste a response that includes the awkward cases.

------------------------------------------------------------------------

## 4. Naming things in Swift

JSON keys are whatever the backend chose. Swift names have rules. The naming code handles a surprising number of cases.

### Properties

`camel()` splits on anything that isn't a letter or digit and camel-cases the result, so `user_name` and `user-name` both become `userName`. A name that starts with a digit gets a leading underscore (`1st` becomes `_1st`). Swift keywords are escaped with backticks rather than renamed, using a list that includes `class`, `default`, `self`, `Type` and the rest:

``` swift
// Real output for {"class": true, "1st": 2, "user-name": "x"}
struct Root: Codable, Sendable {
    let `class`: Bool
    let _1st: Int
    let userName: String

    enum CodingKeys: String, CodingKey {
        case `class`
        case _1st = "1st"
        case userName = "user-name"
    }
}
```

### Types

Nested objects need type names, and the natural source is the key. `pascal()` splits camel-case boundaries too, so `avatarURL` becomes `AvatarURL`. Array element types are singularised with a small rule set plus a table of irregulars:

``` javascript
var IRREG = { people: "Person", children: "Child", men: "Man", women: "Woman", criteria: "Criterion", indices: "Index",
  matrices: "Matrix", news: "News", series: "Series", species: "Species", data: "DataItem", media: "MediaItem",
  status: "Status", statuses: "Status", aliases: "Alias", analyses: "Analysis", quizzes: "Quiz" };
```

So `posts` gives `[Post]`, `categories` gives `[Category]`, `addresses` gives `[Address]` and `people` gives `[Person]`. When no rule applies, the fallback appends `Item` rather than guessing.

There's also a list of **reserved type names**: `Data`, `Date`, `URL`, `String`, `Result`, `Task`, `Locale` and others. A JSON object under the key `data` would otherwise produce `struct Data` and shadow Foundation's `Data` in your module, which is a confusing compile error far away from the cause. The generator appends `Model`, giving `DataModel`.

### One type per shape

If `author` and `editor` have exactly the same keys and types, you want one struct, not `Author` and `Editor`. The generator computes a signature string for every object type (keys, types and optionality, recursively) and reuses the first struct with that signature:

``` swift
// Real output for {"author":{"name":"a","id":1},"editor":{"name":"b","id":2},"data":{"x":1}}
struct Root: Codable, Sendable {
    let author: Author
    let editor: Author
    let data: DataModel
}

struct Author: Codable, Sendable {
    let name: String
    let id: Int
}

struct DataModel: Codable, Sendable {
    let x: Int
}
```

The deduplication is structural only. Two shapes that happen to match but mean different things will share a type; rename it in your code if that reads badly.

------------------------------------------------------------------------

## 5. CodingKeys or convertFromSnakeCase, done properly

The tool offers both key strategies, and the second is subtler than it looks.

With **CodingKeys**, each Swift name maps explicitly to its JSON key, and the enum is only emitted when at least one name differs from its key. With **`keyDecodingStrategy = .convertFromSnakeCase`**, the decoder converts keys itself, and the structs stay short. But it converts them by Foundation's rule, not yours. The tool reimplements that rule:

``` javascript
/* JSONDecoder.KeyDecodingStrategy.convertFromSnakeCase, as Foundation implements it: keep leading/trailing
   underscores, split the rest on "_", lowercase the first part, .capitalized the others (so ID -> Id). */
function fromSnake(key) {
  var m = /^(_*)(.*?)(_*)$/.exec(key), core = m[2];
  if (!core) return key;
  var parts = core.split("_").filter(function (p) { return p !== ""; });
  if (parts.length <= 1) return key;
  return m[1] + parts[0].toLowerCase() + parts.slice(1).map(function (p) {
    return p.charAt(0).toUpperCase() + p.slice(1).toLowerCase(); }).join("") + m[3];
}
```

The surprise for most people is in that last step: `user_ID` becomes `userId`, not `userID`. If you hand-write `let userID` and turn the strategy on, decoding fails with a "key not found" error that looks impossible. Because the generator applies the same rule, the property names it writes are the ones the decoder will look for.

Two edge cases get explicit handling:

- **A key that converts to an invalid Swift name.** The strategy can produce names Swift won't accept. The generator then writes a camel-cased property and a CodingKeys entry whose raw value is the *converted* name, because with the strategy on, that's what the decoder compares against. It adds a note saying so.
- **Two keys that convert to the same name.** `user_id` and `userId` in one object both become `userId` under the strategy, and the decoder can't tell them apart. The generator keeps the first, writes a comment in place of the second, and suggests switching to CodingKeys:

``` swift
// Real output for {"user_id":1,"userId":2} with convertFromSnakeCase
struct Root: Codable, Sendable {
    let userId: Int
    // "userId" left out: it converts to the same key as "user_id"
}
```

With CodingKeys, the same collision is resolved by numbering instead: the second property gets the next free number, and a note records the rename.

------------------------------------------------------------------------

## 6. A decoder snippet that matches the types

Generated types are only half the job. If the structs contain `Date`, the decoder needs a date strategy, and the right one depends on the sample. The generator tracks whether any date had fractional seconds and writes the matching code. For the tool's built-in sample, this is the real output:

``` swift
let decoder = JSONDecoder()
// ISO-8601 with fractional seconds: .iso8601 alone rejects ".123", so use a formatter that allows it
decoder.dateDecodingStrategy = .custom { decoder in
    let string = try decoder.singleValueContainer().decode(String.self)
    let withFraction = ISO8601DateFormatter()
    withFraction.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
    if let date = withFraction.date(from: string) ?? ISO8601DateFormatter().date(from: string) { return date }
    throw DecodingError.dataCorrupted(.init(codingPath: decoder.codingPath, debugDescription: "Bad date: \(string)"))
}
let value = try decoder.decode(Root.self, from: data)
```

The custom strategy tries the fractional format first and falls back to the plain one, so a field that mixes `...00Z` and `...00.123Z` decodes either way. If no date had a fraction, the snippet is just `.iso8601`. If the top level is an array of objects, the snippet decodes `[Item].self`, and a note says so.

If you reuse this in an app, I'd make one change the generator can't make for you: create the formatters once, outside the closure. `ISO8601DateFormatter` isn't free to construct, and the closure runs per date. The generated version keeps everything in one paste-able block, which is the right default for a snippet, not for a hot path.

The page's guide also covers money. Amounts arrive as JSON numbers and the generator types them `Double`, which is correct as a description of the JSON and wrong for arithmetic. Change those properties to `Decimal`; `JSONDecoder` decodes it from a number.

------------------------------------------------------------------------

## 7. Sendable, classes, Hashable and JSONValue

**Sendable is on by default**, because structs whose stored properties are all Sendable value types qualify. The tool won't add it to a `final class` with `var` properties, since the compiler would reject that, and it says why instead of reaching for `@unchecked Sendable`.

**Classes don't get synthesised `Equatable` and `Hashable`**, so when you pick "final class" and tick Hashable, the generator writes `==` and `hash(into:)` over every property. Structs get them synthesised and need nothing.

**Identifiable** is added only to types that actually have an `id` after key conversion.

**JSONValue** is the escape hatch. A field that was only ever `null`, an empty array, or a mix of types can't be typed from the sample, so it becomes a small enum that decodes any JSON:

``` swift
/// Any JSON value. Used for fields whose type the sample doesn't pin down.
enum JSONValue: Codable, Sendable {
    case string(String), number(Double), bool(Bool), object([String: JSONValue]), array([JSONValue]), null

    init(from decoder: Decoder) throws {
        let c = try decoder.singleValueContainer()
        if c.decodeNil() { self = .null }
        else if let v = try? c.decode(Bool.self) { self = .bool(v) }
        else if let v = try? c.decode(Double.self) { self = .number(v) }
        else if let v = try? c.decode(String.self) { self = .string(v) }
        else if let v = try? c.decode([JSONValue].self) { self = .array(v) }
        else { self = .object(try c.decode([String: JSONValue].self)) }
    }
    // encode(to:) mirrors this with a switch
}
```

The enum is only emitted when something uses it, and a note asks you to replace it once you know the real type. That's deliberate. `JSONValue` keeps your code compiling; it shouldn't stay in a finished model.

------------------------------------------------------------------------

## 8. Keeping the page responsive with big responses

The converter runs as you type, with a short debounce: 150 ms normally, 400 ms for large input. Above 200,000 characters, the work moves to a Web Worker so typing doesn't stutter:

``` javascript
var LARGE = 200000, worker = null, job = 0;
function run() {
  var text = input.value, o = opts(), my = ++job;
  if (text.length > LARGE && window.Worker) {
    if (!worker) worker = new Worker(el.getAttribute("data-worker"));
    worker.onmessage = function (e) { if (my !== job) return; if (e.data.ok) show(e.data.result); else fail(e.data.error); };
    worker.postMessage({ text: text, opts: o, core: el.getAttribute("data-core") });
    return;
  }
  try { show(SwiftCodable.generate(text, o)); } catch (e) { fail(e.message); }
}
```

The `job` counter is the important line. If you keep typing while the worker is busy, older results arrive late; `my !== job` throws them away so the page never shows stale output. The worker itself is ten lines: it loads the same core file with `importScripts` and calls the same `generate`. One implementation, two threads.

The JSON is converted in the tab, by the main thread or that worker, and the tool's code doesn't upload it anywhere. For API responses that contain real customer data, that matters.

------------------------------------------------------------------------

## 9. The whole sample, end to end

Press "Try a sample" on the page and you get this JSON (formatted here):

``` json
{
  "id": 42,
  "user_name": "Ana",
  "created_at": "2026-10-25T06:30:00.123Z",
  "avatar_url": "https://example.com/a.png",
  "balance": 12.5,
  "posts": [
    { "id": 1, "title": "Hello", "tags": ["swift"], "likes": 3 },
    { "id": 2, "title": null, "tags": [], "likes": 4, "pinned": true }
  ]
}
```

With default options, the generator produces:

``` swift
struct Root: Codable, Sendable {
    let id: Int
    let userName: String
    let createdAt: Date
    let avatarUrl: URL
    let balance: Double
    let posts: [Post]

    enum CodingKeys: String, CodingKey {
        case id
        case userName = "user_name"
        case createdAt = "created_at"
        case avatarUrl = "avatar_url"
        case balance
        case posts
    }
}

struct Post: Codable, Sendable {
    let id: Int
    let title: String?
    let tags: [String]
    let likes: Int
    let pinned: Bool?
}
```

Every rule from this article is visible: `title` is optional because one post has `null`; `pinned` is optional because one post lacks it; `tags` is `[String]` because the empty array merged with `["swift"]`; `posts` became `Post`; the date has fractional seconds, so the decoder snippet uses the custom strategy from section 6.

Switch "Key mapping" to `convertFromSnakeCase` and the CodingKeys enum disappears, because every key converts to the same names, and the decoder snippet gains `decoder.keyDecodingStrategy = .convertFromSnakeCase`.

------------------------------------------------------------------------

## 10. Before you trust generated models: a checklist

1.  **Paste the messiest real response you have**, with nulls, missing keys and empty arrays. The sample decides optionality.
2.  **Check every `Double` that's really money** and change it to `Decimal`.
3.  **Replace every `JSONValue`** with a real type or a custom `init(from:)`.
4.  **Pick one key strategy per type.** With `.convertFromSnakeCase` on, CodingKeys raw values must be the converted names.
5.  **Remember `user_ID` becomes `userId`** under the snake-case strategy.
6.  **Test the decoder with a fixture** in a unit test, including a date with and without fractional seconds.
7.  **Hoist date formatters** out of the custom strategy closure in production code.
8.  **Read the notes panel.** Duplicate keys, renamed collisions and left-out keys are all reported there.

------------------------------------------------------------------------

## Wrapping up

A JSON-to-Codable generator looks like string templating, but the value is in the decisions: keep number literals, merge array elements into optionals, follow Foundation's real snake-case rule, avoid names that shadow Foundation types, and write a decoder that matches the dates you found. Those are the same decisions I'd make by hand, made the same way every time.

Try it, or browse the related tools and posts:

- **JSON to Swift Codable:** https://bestlatestapps.com/tools/json-to-swift-codable/
- **JSON formatter and validator:** https://bestlatestapps.com/tools/json-formatter/
- **JSON to TypeScript and Kotlin:** https://bestlatestapps.com/tools/json-to-typescript/
- **Convert as you type: currency conversion in Swift:** https://bestlatestapps.com/blog/convert-as-you-type-currency-conversion-in-swift/
- **All of my free web tools:** https://bestlatestapps.com/tools/
- **My portfolio:** https://bestlatestapps.com/alok/

Next time a new endpoint lands in a ticket, paste a real response, not a hand-trimmed one; the converter is only as good as the sample it sees.

*— Alok Kumar Singh*
{% endraw %}
