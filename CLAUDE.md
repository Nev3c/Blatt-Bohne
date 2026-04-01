# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Blatt & Bohne** is a German-language PWA for a specialty café (coffee & tea). The entire frontend is a **single file** (`index.html`) with no build step, no npm, no Babel. React 18 is loaded via CDN and all components use `React.createElement` aliased as `h(...)`.

- **Backend:** Google Apps Script REST API → Google Sheets as database
- **Hosting:** Netlify (static, auto-deploys from GitHub)
- **Live URL:** `heissgetraenke.netlify.app`

## Running / Developing

There is **no build step**. Open `index.html` directly in a browser or deploy all 4 files to any static host.

**Test mode (PIN `1234`):** Loads dummy data, never writes to Google Sheets — safe for development.  
**Admin mode (PIN `4990`):** Full CRUD, real data, polls Sheets every 10s.

To inspect app state: open DevTools → Application → Local Storage.

## Architecture

All state and async logic lives in the single `App` component. Data flows:

```
localStorage (cache) ← → React App state ← → Google Apps Script API ← → Google Sheets
```

### Key state variables
`kaffees`, `tees`, `bestellungen`, `tassen`, `kommentare`, `tab` (kaffee/tee/namen/bestellungen), `isAdmin`, `testModus`

### Important components (all defined in `index.html`)
- `App` — root, all state, all async functions
- `Karte` — beverage card (coffee or tea); handles order, edit, delete, stock toggle
- `Weltkarte` — SVG world map with teardrop pins for origin countries
- `Formular` — add/edit beverage form
- `BestellModal` — multi-step order dialog (preparation → guest name → confirm)
- `PinModal` — SHA-256 PIN authentication
- `CupIcon` — custom SVG cup icon; **must be defined before `TassenBadge`**

### Tab rendering
Uses an IIFE pattern — do **not** replace with ternary chains:
```js
(()=>{
  if (tab === "namen") { return ...; }
  if (tab === "bestellungen") { return ...; }
  return div(null, /* kaffee/tee content */);
})()
```

### World map coordinates
Stored as lat/lon, converted to pixel at render time. Map image is 900×460 equirectangular.

## API Actions (`aktion` parameter to Google Apps Script)
`kaffees_speichern` · `tees_speichern` · `bestellung_hinzufuegen` · `bestellungen_leeren` · `tassen_speichern` · `kommentar_hinzufuegen` · `kommentar_loeschen`

## Auth
PINs verified client-side with Web Crypto API (SHA-256).  
Admin `4990` → `3595dba...` · Test `1234` → `03ac674...`

## Color Constants
```js
const GOLD  = '#c9a96e'  // primary accent
const SAND  = '#d4b483'  // buttons, active state
const BROWN = '#2a1608'  // text on buttons
// Background: #1e120a  Modal: #251508  Card overlay: rgba(61,32,16,0.2)
```

## Data Structures

### Kaffee (Coffee)
```js
{ id, name, subtitle, roest, typ, aromen:[], intensitaet, bild, laender:[], outOfStock,
  params:{ mahlgrad, dosis, temp, zeit, ratio } }
// mahlgrad stored 1–10, displayed ×3 (/30 scale)
```

### Tee (Tea)
```js
{ id, name, subtitle, sorte, aromen:[], intensitaet, bild, laender:[], outOfStock,
  moeglicheMischungen:[{name,id}],  // optional; default: [{name:"pur",id:"m0"}]
  params:{ menge, temp, ziehzeit } }
```

### Bestellung (Order)
```js
{ gast, getraenk, typ, zubereitung, uhrzeit }
// zubereitung: preparation type for coffee; blend name for tea; null for pure tea
```

## Critical Pitfalls

1. **Never change `CACHE_KEY`** — images are stored in Google Sheets keyed to this value; changing it loses all images.
2. **No JSX / no Babel** — use `h(tag, props, ...children)` everywhere; no arrow functions in className expressions.
3. **`serverLaden()` overwrites state completely** — intentional, prevents duplicates; do not merge.
4. **`BestellModal` block needs a trailing `,`** after its last `)` — missing commas here caused many past bugs.
5. **`CupIcon` must be defined before `TassenBadge`** — component ordering matters.
6. **Never use `autoFocus` on inputs** — opens mobile keyboard and obscures the UI below.
