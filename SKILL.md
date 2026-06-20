---
name: ashermaret-website
description: How to maintain, update, and add photos to the ashermaret.com graduation photo slideshow website. Use this skill whenever the user wants to add photos to the site, update an album, fix captions, change the site design, or do anything related to ashermaret.com. Triggers on: "add photos to the site", "update the OHS album", "ashermaret", "Asher's website", "slideshow site", "update the graduation site".
---

# ashermaret.com — Site Maintenance Guide

> **Canonical location of this skill:** committed at the root of the repo
> (`github.com/aaronmaret/ashermaret/SKILL.md`). The repo copy is the source of
> truth. The copy installed in Claude Settings is what Claude auto-loads, so when
> this file changes, re-upload it in Settings so the two stay in sync. If they
> ever disagree, the repo wins.
>
> **Verified accurate against the live repo on Jun 20, 2026.**

## Infrastructure at a Glance

| Item | Value |
|------|-------|
| Live URL | https://ashermaret.com |
| GitHub repo | https://github.com/aaronmaret/ashermaret |
| Branch | `main` (auto-deploys via GitHub Pages, ~30s) |
| DNS | Squarespace → 4 A records (185.199.108–111.153) + CNAME www→aaronmaret.github.io |
| Site file | `index.html` — single file, ~57 KB. All photos, captions, styles, JS, and the audio player live in it. |
| Image hosting | **Directly in this repo**, served via `raw.githubusercontent.com`. (No Google Photos CDN — that approach is deprecated.) |
| Background audio | `Reflection.mp3` in repo root, wired to a 🎵 toggle button (already built). |

## How Operations Are Run

All GitHub operations run through the **GitHub Contents API via `bash_tool`** using
`curl` and Python (`urllib` / `base64` / `json`). Not browser/MCP tools.

The token is assembled from two halves so it isn't stored as one literal string.
**The actual halves are NOT committed to this public repo** (doing so would expose
the token and trigger GitHub secret-scanning auto-revocation). They live in Claude's
memory of this project and in the private Settings copy of this skill. Assembly pattern:

```bash
TOKEN="$(echo -n '<HALF_1>')$(echo -n '<HALF_2>')"
```

Repo: `aaronmaret/ashermaret`, branch `main`.

To decode an API response body in Python:
```python
base64.b64decode(file_data['content'].replace('\n',''))
```

---

## Site Structure

### Four Photo Chapters

Folder → chapter-key → title (verified). Photo counts as of Jun 20, 2026.

| Folder in repo | `chapter` key | Title / subtitle | Count |
|----------------|---------------|------------------|-------|
| `AJM-Slideshow-1-Early` | `early` | Early Years | 41 |
| `AJM-Slideshow-2-Waldorf` | `waldorf` | The Waldorf Years | 20 |
| `AJM-Slideshow-3-Starseeds` | `starseeds` | The Starseeds Era | 30 |
| `AJM-Slideshow-4-OHS` | `ohs` | Owen High School — Warhorse Regiment & Graduation | 41 |

Total: **132 photos.** Each chapter section also has a "View Full Album →" pill button
linking to a Google Photos share URL (those links remain only as external album links —
images themselves are served from this repo, not from Google Photos).

### PHOTOS Array Format

Every photo is one entry in the `PHOTOS` array inside `index.html`:

```javascript
{ src: "https://raw.githubusercontent.com/aaronmaret/ashermaret/main/AJM-Slideshow-4-OHS/HS%20Graduation.jpeg", caption: "HS Graduation", date: "Jun 13, 2026", chapter: "ohs" },
```

- `src` — `raw.githubusercontent.com/aaronmaret/ashermaret/main/<FOLDER>/<filename>`.
  **Spaces in the filename are URL-encoded as `%20`** in the URL.
- `caption` — the filename without extension, with real spaces (e.g. `HS Graduation`).
- `date` — display string in `Mon DD, YYYY` format (e.g. `Jun 13, 2026`).
- `chapter` — one of `early` | `waldorf` | `starseeds` | `ohs`.

### Filename conventions (in the repo folders)

- Use **spaces and literal punctuation** — no underscores. e.g. `Band Awards Anna & Asher.jpeg`, `Anna Asher Hartwell.jpeg`.
- Extension is `.jpeg`.
- The on-disk filename uses real spaces; only the `src` URL encodes them as `%20`.

### Rendering (how index.html builds the page)

`buildGrids()` splits `PHOTOS` by `chapter` into `byChapter`, then renders each into
its `#grid-<chapter>` container. A flat `allPhotos` list drives lightbox navigation.
Empty chapters render a placeholder. So adding a correctly-formed PHOTOS entry is all
that's needed for a photo to appear in the right chapter.

### Audio player (already built — do not re-implement)

At the end of `<body>`:
```html
<audio id="bg-music" loop preload="none">
  <source src="Reflection.mp3" type="audio/mpeg">
</audio>
<button id="music-toggle" onclick="toggleMusic()">🎵</button>
```
`toggleMusic()` plays/pauses and swaps the icon (🎵 ↔ 🔇). Autoplay is intentionally
not forced (browsers block it); the user starts playback with the button.

---

## How to Add Photos to a Chapter

1. **Optimize the image first.** Longest edge ~1600px, JPEG quality ~85, target
   **under 1 MB** (typically 200–800 KB). This keeps it readable through the Contents
   API and fast to load. (See the 1 MB note below.)
2. **Read the local file** from `/mnt/user-data/uploads/`, base64-encode it in Python.
3. **PUT it** to the correct folder via the Contents API:
   `PUT /repos/aaronmaret/ashermaret/contents/AJM-Slideshow-N-.../<filename>.jpeg`.
   A brand-new file needs **no SHA**. Overwriting an existing file needs that file's
   current SHA from a fresh GET.
4. **Verify the upload by bytes, not status.** Fetch the raw URL and confirm
   `content-type: image/jpeg` and a non-zero `content-length`. An HTTP 200 alone does
   not prove the bytes landed.
5. **Re-fetch `index.html`'s current SHA** immediately before editing it.
6. **Insert a PHOTOS entry** after the last existing entry for that chapter. Remember
   to URL-encode spaces in `src`. Ensure the entry you insert *after* ends in a comma.
7. **PUT `index.html`** with the fresh SHA. Confirm the PUT response has `commit.sha`.
8. **Wait ~30s** for GitHub Pages to deploy, then hard-refresh to verify.

### Editing the PHOTOS array safely

- Insert new `ohs` entries after the last existing `chapter: "ohs"` entry.
- A regex that matches an entry with or without a trailing comma:
  ```
  \{ src: "[^"]+", caption: "[^"]*", date: "[^"]*", chapter: "ohs" \},?
  ```
- **A missing trailing comma on the last array entry before an insertion breaks the
  whole site** (all photos vanish). Always verify the comma.
- When rebuilding the full array, preserve all four chapters via separate regex
  matches, then reassemble in order.

### Fixing missing captions

If a photo shows its date where a caption should be, find entries where `caption`
equals `date` and patch in the correct caption (the filename without extension).

---

## Hard-Won Lessons (keep these current)

**GitHub Contents API 1 MB limit.**
`GET /repos/.../contents/<path>` returns `content: ""` for files over 1 MB — metadata
and SHA come back, but no bytes. PUTting that empty content silently creates a 0-byte
file that still returns HTTP 200. For files over 1 MB, pull bytes from
`raw.githubusercontent.com` directly (or re-upload from the local optimized source).
Two current OHS images exceed 1 MB (`Anna Asher Hartwell.jpeg` ~1.76 MB,
`Wintergaurd Percussion.jpeg` ~1.03 MB); they display fine via raw but cannot be
round-tripped through the Contents API GET. Keeping new uploads under 1 MB avoids this
entirely.

**Verify bytes, not just status.** After any upload, check `content-type` and a
non-zero `content-length` — not merely that the status was 200.

**SHA management.** Always re-fetch `index.html`'s SHA immediately before any PUT to
it; a stale SHA causes a 409 conflict. New files need no SHA; overwriting an existing
file (including a 0-byte one) requires the current SHA from a fresh GET.

**CDN caching.** `raw.githubusercontent.com` caches with `max-age=300` (~5 min), and
query strings do not bust it. After fixing a broken file, wait for the `expires` header
time to pass before re-verifying.

**Authoritative size check.** Use the Contents API `size` field, not the raw URL, to
check a file's size.

---

## Design Reference

### CSS Variables
```css
--serif: 'Cormorant Garamond', Georgia, serif
--sans: 'Jost', system-ui, sans-serif
--white: #f8fafc;   --off-white: #f0f4f8;  --stone: #dce6f0
--mid: #93aec7;     --ink: #1a2a3a;        --ink-light: #4a6580
--accent: #2a6496;  --accent-light: #7aafd4
```
Dark navy sections (title, video): `background: #0d1b2e`

### Pill Button (CTAs / album links)
```css
font-family: var(--sans); font-weight: 200; font-size: 0.75rem;
letter-spacing: 0.2em; text-transform: uppercase;
padding: 0.75rem 1.75rem; border-radius: 100px;
border: 1px solid var(--accent-light); color: var(--accent);
```
Hover: `background: var(--accent); color: var(--white);`

---

## Adding a New Chapter

1. Create the new repo folder (e.g. `AJM-Slideshow-5-College`) and upload optimized images.
2. Add a `<section class="chapter-section" id="chapter-NEWKEY">` block following the
   existing pattern (chapter number, title, subtitle, `<div class="photo-grid" id="grid-NEWKEY">`, album-link pill).
3. Add `NEWKEY` to the `grids` map and the `byChapter` object in the render JS.
4. Add the chapter to the nav.
5. Add PHOTOS entries with `chapter: "NEWKEY"`.

Chapter HTML pattern:
```html
<section class="chapter-section" id="chapter-NEWKEY">
  <div class="chapter-header">
    <span class="chapter-number">V</span>
    <div class="chapter-info">
      <h2 class="chapter-title">Chapter Title</h2>
      <p class="chapter-subtitle">Chapter subtitle</p>
    </div>
  </div>
  <div class="photo-grid" id="grid-NEWKEY"><!-- Photos injected by script --></div>
  <div class="chapter-album-link">
    <a href="GOOGLE_PHOTOS_URL" target="_blank" rel="noopener">View Full Album &rarr;</a>
  </div>
</section>
```
