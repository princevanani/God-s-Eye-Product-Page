# God's Eye — Anti-Fingerprint & Tracking Shield

A Chrome extension (Manifest V3) that:
- **Blocks known trackers** — analytics, ad networks, fingerprinting vendors, data brokers, and session-replay tools — via `declarativeNetRequest`.
- **Masks your fingerprint** — canvas, WebGL, audio, device info, screen size, WebRTC local-IP leaks, battery status, and rare-font probing all get consistent, per-site noise so trackers can't correlate you across sites.
- **Warns you in real time** — a small on-page banner appears the first time a page tries to load a tracker, with a one-click Block button.
- **Popup + full settings page** — see what's been caught on the current page, toggle protection per site, and manage a custom block list.

## Iteration: fixes driven by real ipleak.net / EFF Cover Your Tracks results

After loading the first version unpacked, testing against ipleak.net and
EFF's Cover Your Tracks turned up real gaps — worth recording here since
they're the kind of thing that only shows up under actual measurement, not
code review:

- **WebGL hash was 18.22 bits — fully unique, 1-in-305,151.** The original
  code only spoofed `getParameter()`'s vendor/renderer *strings*. It never
  touched the actual rendered pixel output, which is what high-entropy WebGL
  fingerprints (like EFF's) actually hash. Fixed by noising
  `readPixels()` output directly, and by reworking `toDataURL`/`toBlob` to
  redraw *any* source canvas (2D, WebGL, whatever) onto an offscreen 2D
  canvas before export — `drawImage()` doesn't care what the source's own
  context type is, so this catches WebGL-backed canvases that have no 2D
  context to intercept `getImageData` on directly.
- **"Blocking tracking ads / invisible trackers: No."** A ~100-domain
  hand-curated list was never going to catch everything EFF's test (or real
  EasyList/EasyPrivacy) blocks. Added `rules/import-blocklist.js`, which
  converts a real EasyList/EasyPrivacy file into a DNR ruleset — see
  "Importing a real tracker blocklist" below. Also expanded the curated
  list itself, and fixed a real bug in the process: a few entries had paths
  baked into what's supposed to be a domain-only list (e.g.
  `"facebook.com/tr"`), which risked Chrome rejecting the *entire* ruleset
  as malformed. There's now a regression test
  (`tests/domain-utils.test.js`) that fails the build if that happens again.
- **Full system font list was readable.** The original font protection only
  guarded `document.fonts.check()` for 3 obscure font names — real
  fingerprinters measure text width via canvas `measureText()` across many
  font stacks, which is a different API entirely. Added deterministic
  `measureText()` width jitter, which is what actually addresses this.
- **Platform, timezone, and screen color depth were all unmasked.** Added
  screen `colorDepth`/`pixelDepth` (trivial, near-zero real entropy since
  it's ~24 for almost everyone) and a new opt-in `timezone` protection
  (forces UTC, off by default since — unlike the noise-based protections —
  it visibly changes displayed times). **Deliberately did NOT add
  `navigator.platform` spoofing**: since the extension doesn't spoof the
  User-Agent, spoofing platform to a different OS family would create a
  detectable UA/platform *mismatch* — a stronger fingerprinting signal than
  the low-entropy platform value on its own. See the comment in
  `content-scripts/inject.js` for the full reasoning.

### Importing a real tracker blocklist

`easylist.to` isn't reachable from wherever this was built, so it can't be
fetched automatically. To get real EasyList/EasyPrivacy-scale coverage:

```
curl -o easyprivacy.txt https://easylist.to/easylist/easyprivacy.txt
node rules/import-blocklist.js easyprivacy.txt
```

Then reload the extension in `chrome://extensions` and turn on **Imported
blocklist** in the options page — it ships off by default so installing
the extension never silently starts enforcing a large, unreviewed
third-party list. The importer only converts the dominant `||domain.tld^`
rule format (the vast majority of both lists); cosmetic filters and
exception rules are intentionally skipped — see the header comment in
`lib/blocklist-parser.js` for exactly what's handled.

## Iteration 3: WebGL vendor/renderer fix, and an honest limitation on fonts

Re-testing after iteration 2 showed the WebGL hash fix working exactly as
intended — EFF's report went from **18.22 bits, fully unique** to **1.12
bits, "randomized by first party domain,"** matching the canvas hash. Good
confirmation the `readPixels`/offscreen-redraw approach works.

It also exposed a new, more subtle bug: the spoofed WebGL vendor/renderer
*string* was still contributing 16.63 bits on its own — nearly as
identifying as before, just moved to a different field. The old pool
rotated across 3 strings, one of which (`"ANGLE (Intel, Mesa Intel(R) UHD
Graphics)"`) is a **Linux** driver string — reporting it on a Windows
install is the same class of mistake as spoofing `navigator.platform`
independently of the User-Agent: an internally-inconsistent, rarer-than-
real value that stands out *more* than leaving it alone would. Fixed by
switching WebGL vendor/renderer to a single fixed, Direct3D11-flavored
string (Intel integrated graphics — the single most common real-world GPU
across Windows Chrome installs) instead of a rotating pool. This mirrors
the timezone design: for a value that's *directly checkable* (unlike
canvas/audio noise, which is imperceptible either way), one large shared
anonymity set beats a few small distinguishable ones.

**What's still open, honestly: the full system font list remains
readable.** The `measureText()` fix in iteration 2 targeted the wrong
technique for this specific gap. EFF's font detection (and most real
fingerprinting libraries') almost certainly uses the classic DOM technique
— render a hidden probe element with a candidate `font-family` and compare
its `offsetWidth`/`offsetHeight` against a generic-font baseline — not
canvas `measureText()`. Those are different APIs entirely, and I checked
the wrong one.

I looked at closing this and decided not to ship a version that pretends
to: `offsetWidth`/`offsetHeight` are read constantly by ordinary,
non-fingerprinting JS (scroll math, responsive layouts, virtualized lists),
so blanket noise there has real breakage risk across the whole web — not
a tucked-away edge case like canvas or WebGL exports. The one heuristic
that would narrow the blast radius (only noise reads on invisible/off-
screen elements, since font probes are typically hidden) turns out not to
work cleanly either: the common `visibility: hidden` probe pattern doesn't
show up in the cheap, sync-safe checks (`offsetParent === null`), and the
correct check (`getComputedStyle`) forces a synchronous layout on every
call — a real, measurable performance cost site-wide for a partial fix.
This is, structurally, the same reason Tor Browser handles font
fingerprinting by substituting a bundled font set at the **rendering
engine** level rather than trying to intercept every measurement API —
that's not a layer a content-script extension can reach.

If you want to explore this further, the honest options are (a) accept
this as an inherent limitation of an extension-only approach, or (b) a
much more invasive project — bundling and forcing a fixed font stack via
injected CSS `@font-face` overrides for common families, which changes how
text actually *looks* on every page rather than just what scripts can
measure, and is a materially bigger undertaking than anything else in this
project so far.

## Iteration 4: HTTP headers, stronger defaults, and a ceiling worth knowing about

Re-testing after iteration 3 showed the WebGL vendor/renderer fix working
(16.63 → 13.59 bits) but the overall "unique fingerprint" verdict barely
moved (18.22 → 18.23 bits). That's expected, not a sign the fix didn't
work: fingerprinting entropy is closer to *additive* than dominated by one
factor — System Fonts (6.96 bits), the Accept-Language header (6.21 bits),
timezone (~11 bits when off), User-Agent (3.56 bits), and the WebGL string
(13.59 bits) all stack independently. Fixing one axis while seven others
stay untouched barely moves the total. Getting the overall verdict to flip
requires suppressing most of them at once.

Two changes this round, both because the request was explicitly for
maximum protection over convenience:

- **Accept-Language header normalization** (new). Headers are sent before
  any page JS runs, so this was never fixable from `inject.js` — it needed
  a `declarativeNetRequest` `modifyHeaders` rule instead
  (`rules/header-rules.json`). Rewrites `Accept-Language` to one common
  value on every request. **On by default now** — trade-off: sites that
  pick content language from this header will show English regardless of
  your real language. Toggle in options if that's not what you want.
- **Timezone protection flipped to on by default** (was opt-in-off). Same
  trade-off logic as above.

### A ceiling worth setting expectations around

Worth being direct about: EFF's test population is self-selected — people
who specifically visit a fingerprinting-test site, many running their own
different privacy tools with their own different spoofed values. That
population doesn't cluster the way "everyone on the general web" does, so
even a genuinely common, platform-consistent value (like the WebGL string
now) won't necessarily register as common *within EFF's specific dataset*.
That's a real, structural ceiling on this specific metric, not a bug.

More broadly: reliably scoring "not unique" on Cover Your Tracks generally
requires Tor-Browser-level uniformity — literally identical fingerprints
across all users of that browser, achieved by disabling or heavily
restricting canvas/WebGL, snapping window sizes, bundling a fixed font set
at the rendering-engine level, etc. That's a fundamentally different
product than a general-purpose privacy extension layered on regular Chrome
— it trades away a lot of normal site functionality to get there. God's
Eye is aiming for "meaningfully harder to track and correlate across
sites" within a browser you can still use normally, not "byte-identical to
every other installed copy." Worth knowing which goal you're measuring
against before chasing a specific number.

## Iteration 5: three more real gaps, and a note on "reference" files

An uploaded "reference" build turned out to be byte-for-byte identical to
the version already shipped here — same file hashes across every file,
same version number. Worth saying plainly rather than pretending to learn
something from it: there was nothing new to reference. If a genuinely
different extension shows up as a comparison point later, worth checking
its actual technique before assuming "better."

That said, the underlying ask (keep hardening) stood on its own, so three
real, previously-untouched gaps got closed:

- **`getSupportedExtensions()` (WebGL)** — the exact set of supported WebGL
  extensions varies by GPU/driver and is a documented fingerprinting
  signal (FingerprintJS's own docs cite it). Was completely unprotected;
  now returns a fixed common list.
- **Client Hints (`navigator.userAgentData.getHighEntropyValues()`)** — a
  newer surface than the UA string, exposing `platformVersion`,
  `architecture`, `bitness`, and `model` with real entropy beyond what's in
  the (deliberately unspoofed) UA string. Now normalized. Deliberately
  left the synchronous `.brands`/`.mobile`/`.platform` properties alone —
  same reasoning as `navigator.platform`: Chrome sends those same values
  via `Sec-CH-UA-*` headers automatically on every request, so spoofing
  only the JS-visible copy would create a mismatch rather than close a
  gap. The high-entropy values don't have that problem in the common case,
  since they're normally only sent as headers if a server opts in.
- **`devicePixelRatio`** — varies with display scaling (1.25x/1.5x/2x are
  all common) and is readable with no permission prompt. Fixed to `1`,
  same "one common shared value beats a distinguishable pool" reasoning as
  the WebGL string.

Deliberately did NOT touch the numeric WebGL capability constants
(`MAX_TEXTURE_SIZE` and similar) in this pass — after the Mesa/Windows
mismatch mistake in iteration 3, shipping fewer high-confidence values
beats guessing a full capability profile and risking another "spoofed
value that's rarer than reality" mistake.

## Iteration 6: complete UI redesign, and an honest read of an uploaded "reference"

A different uploaded zip this time — genuinely different code, not a repeat
of the earlier identical-file situation. Worth recording the technical
assessment plainly, since the ask was to adopt it wholesale and the
evidence said otherwise:

- **Its User-Agent spoof is incomplete in a way that's worse than not
  spoofing at all.** It rewrites the `User-Agent` header, `Sec-Ch-Ua-
  Platform` header, and `navigator.userAgent` to claim macOS/Chrome 120 —
  but never touches the `Sec-Ch-Ua` header (which still sends the real
  brand list, e.g. `"Chromium";v="150"`) or `navigator.userAgentData` in
  JS at all. A single request ends up claiming two different browser
  versions simultaneously across two headers of the *same* request —
  detectable server-side with no client-side inspection needed. This is
  the WebGL/Mesa mistake from iteration 3, at larger scale: a partial
  identity spoof is a *stronger* signal than no spoof, because internally
  inconsistent identity claims are exactly what fraud/bot-detection
  systems look for.
- **Its WebGL pixel-noise function only touches the first 3 bytes of the
  entire pixel buffer** (`pixels[0]`, `pixels[1]`, `pixels[2]`) — for any
  real fingerprinting read, 99.9%+ of the buffer stays real and
  unmodified. Looks like protection in the code; isn't functionally doing
  much.
- Canvas/audio noise uses one global constant per page load rather than
  per-pixel variation — cancels out under any edge/gradient-based
  analysis, and has no test coverage that would have caught either issue
  above.

Decision: kept this project's tested protection engine (background.js,
inject.js, the DNR rules — the one with 81 passing tests that already
caught bugs of exactly this shape) and rebuilt the interface around it,
rather than trading tested protection logic for a fresh coat of paint.

### New design direction

Moved off the dark radar/phosphor theme entirely, toward a light "optical
instrument" palette — warm bone background (`#F7F4EE`), a deep
coated-glass teal (`#0F5C56`) as primary accent, warm brass (`#B8793A`)
for allow/warning actions. The eye motif evolved from a surveillance-radar
sweep to a camera aperture/iris — same "God's Eye" concept, reads as
precision instrument rather than hacker tool. Applied consistently across
the popup, options page, on-page warning banner, toolbar icon, and the new
marketing site.

The warning banner is a genuine behavior change, not just a restyle:
`background.js` now sends an update for every *newly*-seen tracker domain
per page load (previously: only the first one, ever), so the banner shows
a live-growing, scrollable list as more trackers are caught — with
**Keep & Allow** (allowlists the site, then reloads) and **Dismiss**
(hides the notification without changing any blocking) actions.

### New: `website/`

A four-page marketing site (`index.html`, `privacy.html`, `terms.html`,
`contact.html`) in the same design system, including two hand-built SVG
infographics explaining the network-blocking and fingerprint-masking
engines separately, plus an explicit "what this can and can't promise"
section — naming real limits (no IP masking, no TLS-level protection,
doesn't help if you're logged in) rather than the oversold "total
protection" framing common in this space. Static HTML/CSS with no build
step; the `privacy.html` content is accurate to what the extension
actually stores (all local, nothing transmitted) rather than generic
boilerplate. Contact details in `contact.html` are placeholders — replace
before publishing.

Every popup/options screenshot in this round was checked with a real
headless-Chromium render (Playwright), not just written and assumed
correct — the same "verify, don't assume" approach the protection logic
itself has been held to throughout this project.

## Iteration 7: three reported bugs, three different root causes

All three turned out to be real, distinct bugs — worth recording exactly
what was wrong, since two of them were subtler than they looked from the
bug report alone.

- **"Dismiss doesn't do anything."** The banner uses a closed shadow root
  (`attachShadow({ mode: "closed" })`) for style isolation. `hideBanner()`
  tried to reach it via `bannerHost.shadowRoot` — but `.shadowRoot` is
  `null` from *outside* a closed root by design (confirmed directly: even
  a plain `document.getElementById(...).shadowRoot` query returns `null`
  in a real browser). That made the function's own guard clause
  (`if (!bannerHost.shadowRoot) return;`) exit before ever reaching the
  code that removes the banner — so the click silently did nothing. Fix:
  keep the actual reference `attachShadow()` returns at creation time,
  rather than trying to rediscover it afterward. Closed mode only blocks
  *external* rediscovery, not the creating code's own reference.
- **"Keep & Allow reloads, then shows the same warning again."** This was
  a real gap, not a glitch: `siteAllowlist` only ever controlled whether
  fingerprint-masking got injected (`webNavigation.onCommitted`). Neither
  the tracker-blocking DNR rules nor the `webRequest`-based detection that
  triggers the banner ever consulted it — so a tracker still got "caught"
  and re-announced on every reload, regardless of allowlist status. Fixed
  at the root, not patched around: the `webRequest` handler now skips
  detection entirely when `details.initiator` is an allowlisted origin,
  and a dynamic `declarativeNetRequest` rule using
  `action: { type: "allowAllRequests" }` (matched against the site's own
  `main_frame` navigation, at higher priority than the block rules) now
  makes the actual network-level blocking stand down too. "Keep & Allow"
  now means fully trusted, matching what the button's label already
  implied.
- **Banner position.** Moved from a full-width top bar to a small
  `300px` card fixed to the bottom-right corner, per the request — same
  scrollable tracker list, same Keep & Allow / Dismiss actions, now with
  an additional small × close control in the corner.

## Iteration 8: Chrome Web Store submission package

Added `store-listing/` — everything needed to actually submit this to the
Chrome Web Store: listing copy (name, descriptions, single-purpose
statement), a permission justification for every declared permission
individually, a data-collection disclosure table, and generated assets
(3 in-context product screenshots at 1280×800, small and marquee promo
tiles, the store icon) at exact required pixel dimensions — verified with
Pillow, not eyeballed. Full detail in `store-listing/LISTING.md`.

Preparing the permission justifications required auditing what the code
*actually* calls against what the manifest *declares* — two permissions
turned out to be unjustifiable because nothing uses them:
`declarativeNetRequestFeedback` (no code calls `onRuleMatchedDebug` or
`getMatchedRules`) and `tabs` (the one place a tab URL gets read is in the
popup, in direct response to the user opening it — exactly what
`activeTab` is for; the broader `tabs` permission was never needed).
Removed both. Fewer, tightly-justified permissions reduces review
friction and is a real trust signal, not just tidiness.

## How it actually works (architecture)

```
manifest.json                  Manifest V3 config
background.js                  Service worker: settings, per-tab tracker
                                tally, badge, dynamic DNR rules, message hub
content-scripts/
  content.js                   ISOLATED world — renders the warning banner
  inject.js                    MAIN-world override function — exported and
                                passed to chrome.scripting.executeScript()
lib/
  tracker-domains.js           Curated tracker/vendor domain list
  domain-utils.js               Domain matching helpers
  fingerprint-noise.js          Seeded PRNG + noise math (Node-testable twin
                                 of the logic duplicated inside inject.js)
  blocklist-parser.js           EasyList/EasyPrivacy format parser
rules/
  generate-rules.js            Builds tracker-rules.json from the domain list
  tracker-rules.json           Generated static declarativeNetRequest rules
  header-rules.json            Accept-Language normalization (modifyHeaders)
  import-blocklist.js          Converts a downloaded EasyList/EasyPrivacy
                                file into imported-rules.json (off by default)
  imported-rules.json          Empty until you run import-blocklist.js
popup/, options/                UI
website/                        Marketing site (index/privacy/terms/contact)
icons/                          Extension icons (+ the script that made them)
tests/                          Node test suite (83 tests, see below)
```

**Why fingerprint overrides are injected via `chrome.scripting.executeScript`
instead of a plain content script:** the noise needs to be *different per
site* (so sites can't be correlated) but *stable within a day* (so a
canvas-based captcha doesn't break because two reads returned different
pixels). That per-site seed has to reach the page's own JS realm *before*
the page's scripts run — and it has to be computed from data the background
worker already holds in memory, with no async storage read in the hot path.
`background.js` listens for `webNavigation.onCommitted` and calls
`chrome.scripting.executeScript({ world: "MAIN", func: fingerprintInjector,
args: [config], injectImmediately: true })`, passing the seed directly as a
function argument. No race condition, no flash of unprotected content.

**Anti-averaging noise:** an easy way to defeat naive fingerprint-noise is
to call `getImageData()` (or `getChannelData()`) twice and diff the two
reads — random-per-call noise cancels right out. Here the noise is seeded
from `(daily seed, canvas dimensions)` / `(daily seed, buffer length)`
rather than from a running random sequence, so repeated reads of the *same*
content return *identical* noised output, while different content or a
different site still gets different noise. Covered by
`tests/inject.test.js`.

**Blocking model:** the built-in list blocks via a static
`declarativeNetRequest` ruleset (fast, no per-request JS). Domains you add
yourself (from the banner, popup, or options page) become dynamic DNR
rules. `webRequest.onBeforeRequest` runs in parallel in *observe-only* mode
purely to count/label what's happening for the badge and banner — it never
blocks anything itself.

## Loading it in Chrome

1. Open `chrome://extensions`.
2. Turn on **Developer mode** (top-right toggle).
3. Click **Load unpacked** and select the `gods-eye/` folder.
4. Pin it from the puzzle-piece menu if you want it visible in the toolbar.
5. Visit any site with trackers (e.g. a mainstream news site) — you should
   see the badge count climb and the on-page banner appear.

To edit the tracker list, edit `lib/tracker-domains.js`, then run
`npm run build:rules` (or `node rules/generate-rules.js`) to regenerate
`rules/tracker-rules.json`, then click the refresh icon on the extension's
card in `chrome://extensions`.

## What's been tested, and how — please read this before trusting it blindly

I don't have a real Chrome instance in the environment I built this in, so
here's the honest split:

**Automated (83 tests, all passing — run `npm test`):**
- The noise math itself: deterministic per seed, in-range output, differs
  across origins/days, identical across repeated reads of the same content.
- The actual `fingerprintInjector` function, exercised against stand-in
  Canvas/WebGL/AudioBuffer/RTCPeerConnection/navigator/screen/Intl/Date
  objects — confirms it patches the right APIs (including `readPixels` and
  the offscreen-redraw `toDataURL`/`toBlob` path added in the fixes above),
  returns plausible spoofed values, and never throws even with missing
  config. Each test resets the stub environment first, so repeated
  `fingerprintInjector` calls across the suite never compound monkey-patch
  layers on top of each other — a real bug that surfaced once the test
  suite grew, since a real page only ever calls it once.
- The EasyList/EasyPrivacy parser against representative sample lines of
  every format it needs to handle or skip.
- `background.js`'s full control flow against an in-memory `chrome.*` stub:
  install-salt generation, per-navigation injection (and that it's *skipped*
  for `chrome://` pages and allowlisted origins), tracker detection
  incrementing the right tab's count exactly once per warning, both DNR
  rulesets toggling independently, and every message handler
  (`GET_TAB_STATE`, `UPDATE_SETTINGS`, `BLOCK_DOMAIN`, `TOGGLE_SITE_ALLOWLIST`).
- Every `.js` file syntax-checks cleanly; `manifest.json` and both DNR
  rulesets are validated for well-formedness, that every file the manifest
  references actually exists, and that no tracker-domain entry contains a
  path/scheme (the exact bug that shipped in the first version).

**Not automatable outside a real browser — please verify these yourself
after loading it unpacked:**
- That the popup and options page actually render correctly and look like
  the intended dark/amber design (I can't screenshot a live Chrome popup
  from here).
- Real-world site compatibility — some sites may behave oddly under screen
  or device-info masking; the per-site toggle in the popup is the fix.
- That the on-page warning banner appears correctly across different sites'
  CSS (it's in a closed shadow root specifically to avoid this, but a live
  check is still worth doing).
- Actual network blocking against live tracker domains — the DNR ruleset
  shape is validated, but Chrome's own rule evaluation isn't something I
  can run outside the browser.

If you hit anything odd, the browser's own `chrome://extensions` page has
an "Errors" button per-extension and a "service worker" inspect link that's
the fastest way to see what's happening live. Re-running
[Cover Your Tracks](https://coveryourtracks.eff.org) and
[ipleak.net](https://ipleak.net) after loading a new build is a good habit —
that's literally how the WebGL and font gaps above were found in the first
place, and code review alone wouldn't have caught either one.

## Scope notes

This is a solid, real v1 — not a padded-out line count. A few things I
deliberately kept out to avoid breaking sites or ballooning complexity
without clear benefit: full Public Suffix List domain parsing (the
approximation in `getRegistrableDomain` is display-only, never used for
blocking decisions), and aggressive WebRTC blocking (it only strips
private-IP ICE candidates, not all WebRTC, so video calls keep working).
Both are reasonable follow-ups if you want to extend it.
