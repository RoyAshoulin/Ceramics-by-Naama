# Ceramics by Naama — Project Knowledge

Read this fully before editing. It explains how the site works, why it's built this way, and the rules for changing it.

## What this is

A gallery/catalog site for Naama's handmade ceramics: **www.ceramicsbynaama.co.il** (Hebrew, RTL).
Visitors browse items by category and contact Naama via WhatsApp/Instagram — there is **no cart or checkout**, on purpose. The WhatsApp conversation *is* the shop.

- **Hosting:** GitHub Pages, deployed automatically on every push to `main`. The `CNAME` file holds the custom domain — never delete it.
- **The entire site is one file: `index.html`.** This is deliberate: the owner updates the repo by uploading files through the GitHub web UI, so a single file keeps that workflow simple. **Do not split it into separate CSS/JS files without explicit approval.**

## Working rules (important)

1. **Behavior-preserving fixes** (bugs, performance, code cleanup) may be applied directly.
2. **Anything that changes the interface, look, or user-facing behavior must first be proposed as a numbered list** — Roy picks which items to implement. Never restyle or add UX features unprompted.
3. Keep the minimal, warm aesthetic. Colors live in CSS variables (`:root` — `--bg`, `--text`, `--muted`, `--accent`, `--border`).
4. Pushing to `main` **is** deploying to the live site. Verify locally first; let Roy decide when to push unless he says otherwise.

## Architecture: Google Drive is the CMS

Naama manages all content by uploading images to Google Drive folders — she never touches code.

- **Folder = stock level.** Five folders (IDs in the `CONFIG` block at the top of `index.html`): `"1"`, `"2"`, `"3"`, `"4"`, `"sold"`. Moving an image between folders changes the displayed stock. Moving it to `sold` grays the card and hides the contact buttons.
- **Filename = item data:** `name(imageNumber)_price_tag.jpg`, in Hebrew.
  Example: `ואזה ספארי קוטר 10(2)_120_ואזות.jpg` → item "ואזה ספארי קוטר 10", photo #2, ₪120, category "ואזות".
  - The `(n)` part is optional; files sharing the same `name_price_tag` are grouped into one item with multiple photos (crossfade on the card, arrows/dots/swipe in the lightbox).
  - Hyphens in the name part become spaces. A missing/non-numeric price shows no price.
- **Categories (tags) are derived** from the third filename segment — there is no hardcoded category list. A new tag in a filename automatically creates a dropdown entry and a home-page tile.
- **The "Logo" folder** (`logoFolderId`) holds special files matched by exact base name (case-insensitive): `Logo` (home splash logo), `Splash image` + `Text` (story page), `wa`, `ig`, `mail` (contact icon images).
- Images render via `https://drive.google.com/thumbnail?id=<fileId>&sz=w800` (w1200/w1600 for logo/story).
- The Drive `files.list` query includes `pageSize=1000` — the API default of 100 silently drops files beyond that (folder "1" had 85 files in July 2026, so this was a real, near-miss bug). **1000 files per folder is the hard ceiling**; pagination would be needed beyond that.

## The API key (safety)

`CONFIG.apiKey` is a **browser API key — public by design** and safe to keep in the HTML **because it is HTTP-referrer-restricted** to the site's domain in Google Cloud Console (verified: no referrer → 403, site referrer → 200).

- Consequence: **the Drive API always returns 403 from localhost.** This is expected, not a bug.
- If the key is ever rotated, the new key must get the same referrer restriction (and ideally be restricted to the Drive API only).

## How to test locally

Because of the referrer restriction, local pages can't fetch real data. The workflow that works:

1. Serve the folder statically (`.claude/launch.json` defines a `static-site` config: `python -m http.server 8642`).
2. The page will show the error state ("הגלריה לא נטענה כרגע") — expected.
3. **Inject mock data from the console** and drive the UI directly:
   ```js
   allItems = [{ name:'קערה', price:150, tag:'קערות', units:'2',
                 imgs:['data:image/svg+xml,...'], img:'data:image/svg+xml,...' }];
   buildDropdown();
   showTag('קערות');
   ```
   Data-URI SVGs work well as fake images. This exercises everything: cards, sorting (sold last), lightbox, history, prices.
4. To verify real Drive data, request the API URL with a `Referer: https://www.ceramicsbynaama.co.il/` header (e.g. from PowerShell/curl).

## Code map (all inside index.html)

- **`CONFIG` block** (top of `<body>`) — the only section Naama-facing settings live in: artist name, email, WhatsApp number, Instagram URL, site URL, Drive folder IDs, API key.
- **Views** — plain `display` toggling, no framework: `#splash` + `#tag-gallery` (home), `#story-view`, `#shop-view`. Switch with `showSplash()` / `showStory()` / `showTag(tag)`.
- **History routing** — `syncHistory()` pushes `#story` / `#gallery/<tag>` hashes; `popstate` → `renderFromHash()` re-renders without pushing (guarded by `suppressHistory` and `currentViewKey`). Hash URLs deep-link on load (handled at the end of `init()` after data arrives). Keep this invariant: *view functions render; syncHistory is the only thing that touches history.*
- **Data flow** — `init()` → `loadAllItems()` fetches the five folders with `Promise.allSettled` (one failing folder must never blank the whole gallery) → `parseFilename()` → grouped by `name||price||tag||units` → `allItems` → `buildDropdown()` builds the dropdown and home tiles.
- **Cards** — `makeCard()`. Multi-photo cards crossfade via `setInterval`; every timer is pushed to `cardTimers` and `clearCardTimers()` runs on each grid rebuild — **keep this pairing or timers leak.**
- **Lightbox** — `openLightbox()` etc.; supports arrows, dots, keyboard (Escape/←/→), and horizontal swipe. WhatsApp button builds a prefilled Hebrew message with the item name + site URL.
- **`esc()`** — item names come from Drive filenames and are HTML-escaped wherever they're placed in `innerHTML`. Use it for any new injection point.
- **Fonts** — Assistant loads from Google Fonts; "Parmigiano Sans" is first in the stack but only resolves if locally installed, so Assistant is what visitors actually see.

## Ideas proposed but not (yet) approved

Deliberately not implemented — propose again rather than just building:
"הכל" (all items) view · hide/separate sold items · pinned category cover images (random item per tile today) · 2-column tile grid on mobile · favicon + Open Graph tags for link previews · sessionStorage caching of Drive listings (tradeoff: delays new uploads appearing) · splitting index.html into multiple files.
