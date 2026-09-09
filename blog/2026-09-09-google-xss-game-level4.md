---
slug: google-xss-game-level4
title: 'Google XSS Game – Level 4: Escaping an onload Handler'
authors: [tobias]
tags: [hacking, ctf, xss, google-xss-game]
---

A walkthrough of **Level 4** of the [Google XSS Game](https://xss-game.appspot.com/level4) — popping an `alert()` by breaking out of a JavaScript string that gets concatenated directly into an HTML event handler attribute.

<!-- truncate -->

**Category:** Cross-Site Scripting (OWASP Top 10: A03:2021 – Injection)

## Goal

Get `alert()` to fire on the page, using a URL that only differs in its query parameters.

## Step 1: Find the Source

I started by looking at the page as a black box: what user-controllable input does it consume? The page shows a countdown timer, and the value that seeds it comes straight from the URL — a query parameter that sets how many seconds the timer counts down. That parameter is the **source**.

## Step 2: Investigate `startTimer()`

Since that parameter feeds straight into the countdown, I first went looking for a fitting sink inside the `startTimer()` function itself — i.e. whether the vulnerability lived in the function's own logic. Looking at its implementation, the first line is:

```js
seconds = parseInt(seconds) || 3;
```

## Step 3: Rule Out `startTimer()` Itself as the Sink

This line kills the theory that the function itself is the sink: whatever string gets passed in is forced through `parseInt()`, and anything that doesn't parse to a number falls back to `3`. No matter what I pass as the argument, by the time it's used *inside* the function it has already been coerced into a harmless number. So the function body can't be the sink — any payload would just get parsed away, and validating/coercing input inside a downstream function like this is useless if the vulnerability actually sits somewhere else entirely.

## Step 4: Realize the `onload` Attribute Could Be the Sink

That sent me looking further upstream: not at how `startTimer()` processes its input, but at how the page builds the call to it in the first place. Instead of being assigned safely (e.g. via a property or `textContent`), the raw parameter is dropped straight into the markup that starts the timer, ending up inline inside an `onload` handler roughly like this:

```html
<img src="/static/loading.gif" onload="startTimer('<user input>')">
```

The parameter value is spliced directly between the quotes of the `startTimer('...')` call, with no escaping — and, critically, *before* `parseInt()` ever gets a chance to run. Since it lands inside a JavaScript context (an inline event handler), and that context is built by naive string concatenation, this is a classic injection point: anything that closes the surrounding quote and call is executed as real JavaScript. This is the actual sink.

## Step 5: Break Out of the String

With the sink identified, the exercise became: craft a value that closes `startTimer('` cleanly and then runs my own code. A few of the payloads I tried first:

```
"; alert(`hack`)"
'); alert(`hack`)
```

Neither worked. The problem is the template still appends its own closing `')` *after* whatever I inject — so whatever I write has to leave the tail end of the attribute (`')`) as syntactically valid JavaScript, not stray characters that break parsing. My payloads either left the original string unclosed (so my code was just interpreted as harmless data) or left a dangling quote/parenthesis at the end that produced a syntax error and killed the whole handler before `alert()` could run.

## Step 6: The Working Payload

The payload that finally worked:

```
');alert('hack
```

The key insight: instead of fighting the template's trailing `')`, I let it close *my* injected call. Here's how the final attribute assembles, character by character:

```
startTimer('  +  ');alert('hack  +  ')
= startTimer('');alert('hack')
```

- `startTimer('` — the original, untouched prefix from the template.
- `');alert('hack` — my payload: first closes the original string and call (`'`, `)`, `;`), then opens a fresh call to `alert(` with an intentionally **unterminated** string `'hack`.
- `')` — the template's own suffix, which I no longer fight against. It supplies the closing quote for `'hack` and the closing parenthesis for `alert(...)`.

The result is perfectly valid JavaScript — `startTimer('');alert('hack')` — and the alert fires.

## Key Takeaways

- Validation or coercion (like `parseInt(seconds) || 3`) inside a *downstream* function doesn't protect against injection that happens *upstream*, at the point where the raw input is embedded into markup. Always trace the input to where it's first written into HTML/JS, not just to where it's ultimately consumed.
- Concatenating user input directly into an inline event handler (`onload="...('<input>')"`) is just as dangerous as concatenating it into a `<script>` block — it's still a JavaScript context.
- An injection payload doesn't have to be self-contained. If the sink appends a fixed suffix after your input, you can deliberately leave a string or call unterminated and let that suffix close it for you.
- When a naive payload fails, check whether it left the surrounding code in a syntactically broken state (killing the whole handler) versus simply being swallowed as inert data — the fix is usually to balance quotes/parens with what the template appends, not to add more punctuation.

## Security Risk & Impact

This is textbook DOM-based / reflected XSS caused by unsafe string interpolation into an executable HTML attribute. In a real application, any input reflected this way lets an attacker run arbitrary JavaScript in the victim's browser session — stealing cookies or tokens, performing actions as the logged-in user, defacing the page, or pivoting into further attacks (e.g. credential phishing overlays). The fix is never to build event handlers or script content via string concatenation of user input; instead, values should be passed through safe DOM APIs (`element.addEventListener`, setting `.textContent`/properties) or, if templating is unavoidable, properly context-aware encoded for the JavaScript-in-attribute context.
