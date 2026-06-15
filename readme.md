# LISTN — Emotions as Visual Language

LISTN is a front-end web platform that translates human emotions into generative artworks. Users describe how they feel in their own words (in English or Spanish), and the system maps that description to one of four emotional categories, then selects and displays an image from a curated bank that visually represents that feeling.

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Page Architecture — HTML & CSS](#3-page-architecture--html--css)
4. [How Pages Connect to Each Other](#4-how-pages-connect-to-each-other)
5. [The Emotion Classification System](#5-the-emotion-classification-system)
6. [Image Bank Organisation (ImageKit)](#6-image-bank-organisation-imagekit)
7. [How Images Are Fetched and Displayed](#7-how-images-are-fetched-and-displayed)
8. [User Workflow](#8-user-workflow)
9. [Design System & Visual Style](#9-design-system--visual-style)
10. [Responsive Behaviour](#10-responsive-behaviour)

---

## 1. Project Overview

LISTN converts emotional language into visual form. The core idea is that the act of articulating a feeling (even partially) can be met with an artwork that reflects it back. The platform is built entirely in vanilla HTML, CSS, and JavaScript with no frameworks or build tools. All assets are served from the ImageKit CDN.

**Core capabilities:**

- Emotion input via free text (English or Spanish) or one-click pills
- Two-stage classification: category detection - image sub-range selection
- Generative canvas overlay layered on top of the selected photograph
- Feed of community pieces, profile page, share flow, and sign-in page
- Fully responsive across desktop, tablet, and mobile

---

## 2. Repository Structure

````
LISTN/
│
├── index.html        Main landing page — hero, feed grid, create section, system section
├── index.css         All styles for index.html
│
├── profile.html      User profile — works grid, saved grid, about tab, edit modal
├── profile.css       All styles for profile.html
│
├── share.html        Share a piece — artwork panel, friends selector, copy link, download
├── share.css         All styles for share.html
│
├── signin.html       Sign-in page — email/password form, Google and Apple buttons
├── signin.css        All styles for signin.html
│
└── readme.md         This file


Each HTML page has one dedicated CSS file. There are no shared stylesheets, each page is self-contained. The CSS file is linked in the `<head>` of its HTML file:

```html
<link rel="stylesheet" href="index.css" />
````

## 3. Page Architecture — HTML & CSS

### `index.html` + `index.css`

The main page. Contains five sections stacked vertically:

| Section | ID        | Description                                                 |
| ------- | --------- | ----------------------------------------------------------- |
| Hero    | `#top`    | Full-viewport title and subtitle                            |
| Ticker  | —         | Scrolling marquee of emotion labels                         |
| Feed    | `#feed`   | 12-column asymmetric image grid (21 pieces)                 |
| Create  | `#create` | Emotion textarea + pills + generate button + canvas preview |
| System  | `#system` | 4 clickable category cards with expandable image gallery    |
| Footer  | —         | Copyright line                                              |

The CSS file (`index.css`) covers all layout, animations, the custom cursor, the detail panel (slide-up overlay on grid item click), the generative canvas preview area, the system gallery, and all responsive breakpoints.

### `profile.html` + `profile.css`

The user profile page. Divided into:

- **Header** — avatar, name, handle, bio, stats (pieces / followers / following), action buttons
- **Tabs** — Works (9 images, complex 12-column grid), Saved (6 images, 3-column grid), About (bio + emotional signature bars)
- **Edit Profile modal** — opens on button click, saves name/handle/bio to the DOM
- **Footer**

Images in the Works and Saved grids are loaded from ImageKit via JavaScript on page load using `data-emotion` and `data-num` attributes on each item.

### `share.html` + `share.css`

A split-layout page that appears when the user shares a piece:

- **Left panel** — full-height artwork display (loads from `localStorage` — either an ImageKit URL from the profile or a base64 canvas from the generator)
- **Right panel** — piece title/author/emotion metadata, copy link input, friends selector (6 clickable cards), social share buttons, download button

Data is passed between pages via `localStorage` under the key `listn_share_piece`.

### `signin.html` + `signin.css`

A split-layout sign-in page:

- **Left panel** — editorial headline, tagline, emotion category tags (hidden on mobile)
- **Right panel** — email + password fields with inline validation, submit button, Google and Apple social buttons

On successful submission the page redirects to `profile.html` after 1.2 seconds.

---

## 4. How Pages Connect to Each Other

```
index.html
  │
  ├── Nav → profile.html
  ├── Nav → signin.html
  ├── Grid item click → detail panel (on same page)
  ├── "Share with friends" button in detail panel
  │       └── saves piece to localStorage → navigates to share.html
  │
  └── Works grid item click (profile.html)
          └── saves piece to localStorage → navigates to share.html

profile.html
  ├── Nav → index.html (feed, create)
  ├── Nav → signin.html
  ├── "Create piece" button → index.html#create
  └── Work item click → share.html (via localStorage)

share.html
  ├── Nav back arrow → index.html
  ├── Nav → index.html (feed, create)
  └── Nav → profile.html

signin.html
  ├── Nav → index.html (feed, create)
  └── Submit / social buttons → profile.html
```

**Cross-page data (localStorage key: `listn_share_piece`)**

```json
{
  "emotion":  "ambiguity",
  "title":    "Between Worlds",
  "author":   "@julia_c",
  "time":     "just now",
  "imageUrl": "https://ik.imagekit.io/..."   // when shared from profile
  // OR
  "image":    "data:image/png;base64,..."    // when shared from generator
}
```

---

## 5. The Emotion Classification System

The system works in two stages every time the user generates a piece.

### Stage 1 — Category Detection (`classifyEmotion`)

The user's free-text input is normalised (lowercased, diacritics stripped so Spanish accented characters like á, é, ñ match their unaccented equivalents) and then scored against four keyword maps:

| Category      | Colour           | Example triggers (EN)                 | Example triggers (ES)                    |
| ------------- | ---------------- | ------------------------------------- | ---------------------------------------- |
| **Intensity** | Red `#c0392b`    | rage, terror, love, euphoria, panic   | rabia, miedo, amor, euforia, panico      |
| **Calm**      | Blue `#9ed5de`   | sad, lonely, serene, melancholy, numb | triste, solo, tranquilo, melancolico     |
| **Balance**   | Green `#265e55`  | grateful, joy, clarity, hope, peace   | alegria, feliz, gratitud, esperanza      |
| **Ambiguity** | Purple `#7b5ea7` | nostalgia, wonder, confused, torn     | nostalgia, asombro, confundido, dualidad |

A two-tier weighting system prevents false ties:

- **Weight 2** — highly distinctive words (e.g. `rage`, `euphoria`, `nostalgia`, `soledad`)
- **Weight 1** — common emotional words that may appear in multiple contexts

The category with the highest total score wins. If no keyword matches, one of the four is chosen at random.

### Stage 2 — Sub-Range Selection (`pickImageIndex`)

Each category is divided into four emotional sub-ranges, each corresponding to a band of image numbers in the ImageKit folder. The normalised input is scored against each range's tag list, and the best-matching range is selected. A random image number within that range is then picked.

**Intensity (images 00 – 23)**

| Range   | Images               | Emotional states                                        |
| ------- | -------------------- | ------------------------------------------------------- |
| 00 – 06 | Passionate / love    | passionate, euphoric, electric, alive, love, excitement |
| 07 – 12 | Anger / rage         | anger, rage, urgency, explosive, confrontational        |
| 13 – 19 | Terror / panic       | terror, panic, dread, dark fury, violent intensity      |
| 20 – 23 | Grief-rage / despair | grief-rage, despair, burned out, imploding              |

**Calm (images 00 – 31)**

| Range   | Images             | Emotional states                               |
| ------- | ------------------ | ---------------------------------------------- |
| 00 – 07 | Serene / peaceful  | serene, peaceful, dreamy, gentle, safe, tender |
| 08 – 13 | Melancholy / sad   | melancholy, wistful, sad, longing, drifting    |
| 14 – 21 | Lonely / hollow    | lonely, hollow, numb, empty, disconnected      |
| 22 – 31 | Dissociated / flat | dissociated, flat, exhausted, fading           |

**Balance (images 00 – 32)**

| Range   | Images                  | Emotional states                                 |
| ------- | ----------------------- | ------------------------------------------------ |
| 00 – 07 | Joy / happiness         | joy, happiness, lightness, optimism, hope        |
| 08 – 17 | Contentment / gratitude | contentment, gratitude, warmth, grounded, stable |
| 18 – 24 | Clarity / focus         | clarity, focus, purpose, structured calm         |
| 25 – 32 | Acceptance / peace      | acceptance, okayness, humble peace, just enough  |

**Ambiguity (images 00 – 38)**

| Range   | Images               | Emotional states                                                        |
| ------- | -------------------- | ----------------------------------------------------------------------- |
| 00 – 06 | Wonder / awe         | wonder, awe, curiosity, magic, dreamlike                                |
| 07 – 17 | Nostalgia / memory   | nostalgia, bittersweet, memory, longing for the past                    |
| 18 – 28 | Duality / conflict   | duality, contradiction, conflict, torn between two things               |
| 29 – 38 | Grief-love / unknown | grief-love, dissociation, impossible longing, "I don't know how I feel" |

### Multilingual support

The same keyword maps and sub-range tag lists contain both English and Spanish terms side by side. Before any comparison, both the user's input and the keyword are passed through `normalizeText()`:

```js
function normalizeText(str) {
  return str.toLowerCase().normalize("NFD").replace(/[̀-ͯ]/g, "");
}
```

This means `"emoción"`, `"emocion"`, and `"EMOCIÓN"` all match the same keyword. Users can type in either language or mix them freely.

---

## 6. Image Bank Organisation (ImageKit)

All images are hosted on [ImageKit](https://imagekit.io) under the project ID `qppkb7d3b`. The folder structure mirrors the four emotion categories exactly:

```
https://ik.imagekit.io/qppkb7d3b/
├── intensity/
│   ├── intensity_00
│   ├── intensity_01
│   ├── ...
│   └── intensity_23
├── calm/
│   ├── calm_00
│   ├── ...
│   └── calm_31
├── balance/
│   ├── balance_00
│   ├── ...
│   └── balance_32
├── ambiguity/
│   ├── ambiguity_00
│   ├── ...
│   └── ambiguity_38
└── logo/
    └── Logo TFG.svg
```

File naming convention: `{emotion}_{number}` where the number is always zero-padded to two digits (e.g. `intensity_07`, `calm_14`, `balance_03`). There are no file extensions in the URLs — ImageKit serves the appropriate format automatically via the `f-auto` transform parameter.

---

## 7. How Images Are Fetched and Displayed

### URL construction

A single utility function builds every ImageKit URL used across the site:

```js
const IK_BASE = "https://ik.imagekit.io/qppkb7d3b";
const IK_PARAMS = "?tr=q-85,f-auto,w-1400"; // feed / share
// profile uses w-800 instead

function ikUrl(emotion, num) {
  const n = String(num).padStart(2, "0");
  return IK_BASE + "/" + emotion + "/" + emotion + "_" + n + IK_PARAMS;
}
```

Transform parameters applied to every request:

- `q-85` — 85% quality (balance of sharpness and file size)
- `f-auto` — ImageKit automatically serves WebP to browsers that support it, JPEG otherwise
- `w-1400` or `w-800` — image is resized server-side to the appropriate width before delivery

### Feed grid (index.html)

The first 9 grid items contain inline base64-encoded JPEGs as fallbacks. Items 10 – 21 have empty `<img class="grid-img" alt="">` tags; JavaScript fills in their `src` at page load using `ikUrl()` with pre-chosen index numbers from the `gridSamples` map:

```js
const gridSamples = {
  intensity: [0, 3, 8, 15, 20],
  calm: [0, 2, 10, 18, 25],
  balance: [0, 5, 12, 20, 28],
  ambiguity: [0, 4, 14, 25, 33],
};
```

### Profile grids

Work and Saved items each carry `data-emotion` and `data-num` HTML attributes. On page load, JavaScript reads those attributes and assigns the `src`:

```html
<div class="work-item" data-emotion="ambiguity" data-num="22">
  <img class="work-bg" alt="Between States" />
</div>
```

```js
document.querySelectorAll(".work-item").forEach((item) => {
  item.querySelector(".work-bg").src = ikUrl(
    item.dataset.emotion,
    item.dataset.num,
  );
});
```

### System gallery (index.html)

When a user clicks one of the four category cards, JavaScript builds a 6-image gallery on the fly. Each image is fetched via `ikUrl()` with hand-picked index numbers that represent each sub-range within the category:

```js
const galleryData = {
  intensity: [
    {
      num: 2,
      label: "Passion",
      title: "Burning Edge",
      author: "@marta_v",
      time: "2h ago",
    },
    {
      num: 5,
      label: "Euphoria",
      title: "Electric Rush",
      author: "@felix_r",
      time: "5h ago",
    },
    // ...
  ],
  // calm, balance, ambiguity follow the same structure
};
```

### Generative canvas overlay

After the image loads, a `<canvas>` element is drawn on top of it with a procedurally generated overlay — particles, shapes, and a grain texture — using parameters defined per emotion category in `EMOTION_META`. The overlay colours are drawn from the `PALETTE_FAMILIES` map, which contains five distinct colour palettes per category. The final result is a composite of the photograph and the generative layer.

---

## 8. User Workflow

```
User lands on index.html
  │
  ├── Scrolls the feed → clicks a piece → detail panel slides up
  │     └── Clicks "Share with friends" → share.html
  │
  └── Scrolls to Create section
        │
        ├── Types emotion in textarea (English or Spanish)
        │     OR clicks an emotion pill to pre-fill
        │
        └── Clicks "Generate Artwork"
              │
              ├── classifyEmotion() → determines category
              ├── pickImageIndex()  → determines image number within category
              ├── ImageKit URL is built and image is fetched
              ├── Canvas overlay is drawn on top
              │
              └── Preview section appears (full-screen)
                    ├── "Regenerate" → picks new image with same emotion
                    ├── "Share with friends ↗" → saves to localStorage → share.html
                    └── Detail panel (if opened from feed) shows "Share" button
```

---

## 9. Design System & Visual Style

All pages share a consistent set of CSS custom properties defined in `:root`:

```css
:root {
  --red: #c0392b; /* Intensity */
  --blue: #9ed5de; /* Calm */
  --green: #265e55; /* Balance */
  --purple: #7b5ea7; /* Ambiguity */
  --black: #1f1f22; /* Background */
  --white: #f9f6f0; /* Text / UI */
  --grey: #2e2e32; /* Borders / cards */
  --mid: #888; /* Secondary text */
  --dim: #444; /* Tertiary / disabled */
}
```

**Typography** — Helvetica Neue, fallback to Helvetica. Headings are uppercase, weight 900, with negative letter-spacing for tight editorial feel. Body copy is weight 400 at 13–14px.

**Custom cursor** — A 10px dot (`mix-blend-mode: difference`) paired with a 38px lagging ring, both `position: fixed`. On hover over images the dot expands. On the profile page the dot takes the emotion's category colour when hovering a work.

**Grid** — All main content areas use a 12-column CSS Grid. The feed grid uses varied `grid-column` spans and `aspect-ratio` values to create an asymmetric editorial layout. On mobile it collapses to 2 columns, then 1 column.

**Animations** — Scroll-reveal via `IntersectionObserver` (adds `.visible` to `.reveal` elements). Page-load staggered `fadeUp` keyframe. Hover states use cubic-bezier easing (`0.25, 0.46, 0.45, 0.94`).

**Noise texture** — A fixed SVG fractal noise overlay at 30% opacity sits on top of every page via `body::before`, giving the dark backgrounds a subtle grain.

---

## 10. Responsive Behaviour

All four pages include responsive CSS at two breakpoints:

| Breakpoint         | Target          | Key changes                                                                                                              |
| ------------------ | --------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `max-width: 768px` | Tablet / mobile | Nav links hidden → hamburger menu appears; grids collapse to 2 columns; split layouts stack vertically; paddings reduced |
| `max-width: 480px` | Small mobile    | Grids collapse to 1 column; further padding reductions; social buttons stack vertically                                  |

**Mobile navigation** — A hamburger button (3-line icon) replaces the nav link row on screens ≤ 768px. Clicking it opens a full-screen overlay with large uppercase links. The icon animates into an × on open.

**Touch devices** — The custom cursor is hidden using `@media (hover: none) and (pointer: coarse)`, and `cursor: auto` is restored so the OS default pointer is used instead.

**Share page** — The left/right split becomes top/bottom: artwork panel takes ~55vh, share panel scrolls below. Friends grid goes from 3 to 2 columns at 480px.

**Sign-in page** — The left editorial panel is hidden on mobile; only the form is shown, taking full width.
