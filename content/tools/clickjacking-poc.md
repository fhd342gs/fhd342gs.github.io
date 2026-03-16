---
title: "Clickjacking PoC Generator"
date: 2026-03-16
draft: false
tags:
  - clickjacking
  - web-security
  - poc
  - x-frame-options
  - csp
categories:
  - tools-and-techniques
description: "Quick and dirty clickjacking PoC generator for testing X-Frame-Options and CSP frame-ancestors."
---

## What's clickjacking

Target page gets loaded in an invisible iframe. Overlay it on a decoy page. Victim thinks they're clicking "Free iPad" but they're actually clicking "Delete My Account" on the real site underneath.

It's been around forever and the fix is trivial, which makes it surprising how often it's still missing.

---

## The fix (and what we're testing for)

Two headers kill this attack:

```
X-Frame-Options: DENY
```
or
```
Content-Security-Policy: frame-ancestors 'none'
```

`DENY` blocks framing entirely. `SAMEORIGIN` / `frame-ancestors 'self'` allows same-origin framing only. If neither header is present, the page can be framed by anyone.

---

## The tool

Dead simple. An HTML page that iframes a target URL.

The core concept is literally this:

```html
<iframe src="https://TARGET_URL/" height="500" width="800"></iframe>
```

If the target renders inside the frame — vulnerable. If you get a blank frame or an error — headers are working.

The PoC generator wraps this in a nicer UI where you punch in a URL and it builds the test page for you. Nothing fancy. Does what it needs to do.

---

## Usage

1. Open the HTML file in a browser
2. Enter the target URL
3. Page loads in the iframe? **Vulnerable.**
4. Blank frame or broken? **Protected.**

That's it. You now have a visual PoC you can screenshot for a report.

---

## When to report it

This is the part most people get wrong.

{{< callout type="warning" >}}
A missing `X-Frame-Options` header is not automatically a finding. Stop reporting it like one.
{{< /callout >}}

**Report it when:**
- The frameable page has **sensitive actions** — password change, fund transfer, settings modification, OAuth authorization
- You can demonstrate a realistic attack scenario where a user would be tricked into performing an unintended action
- The application relies on click-based actions that don't have additional confirmation steps (CSRF tokens alone don't prevent clickjacking)

**Don't report it when:**
- The page is a static marketing site with no interactive elements. "Your homepage can be framed" is not a vulnerability, it's a fact about HTML.
- It's a login page with no other sensitive actions. Low impact, everyone knows, nobody cares.
- Your scanner flagged "missing X-Frame-Options" on a page that displays a logo and a phone number. Congratulations, you found nothing.
- The page already uses `frame-ancestors` in CSP but your tool only checked for `X-Frame-Options`. Read both headers before filing.

{{< callout type="note" >}}
The difference between a useful pentest report and noise is knowing when a technical finding has actual impact. Clickjacking on a page where the worst outcome is "user unknowingly views a product page" is not worth anyone's time to read, triage, or fix.
{{< /callout >}}

---

## Source

Code is on GitHub: [fhd342gs/Clickjacking_PoC](https://github.com/fhd342gs/Clickjacking_PoC)
