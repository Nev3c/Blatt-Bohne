# Blatt & Bohne — Projekt-Dokumentation

## Stack & Deployment
- **Single-file PWA:** `index.html` (~297KB), `manifest.json`, `icon.png`, `apps_script.js`
- **Frontend:** React via CDN, `React.createElement` (kein Babel, kein npm!)  
  `const h = React.createElement` — alle Komponenten mit `h(...)` aufgerufen
- **Backend:** Google Apps Script als REST-API → Google Sheets als Datenbank
- **Hosting:** GitHub → Netlify (alle 4 Dateien droppen)
- **API URL:** `https://script.google.com/macros/s/AKfycbwopM00SKmUf324x_Z6j0iZLfvxwWZfrJOzun0e8jN7a3udzECXJ744hjPIBKmaASbY3Q/exec`
- **CACHE_KEY:** `dennis_v18` — NIEMALS ändern (Bilder in Sheets gespeichert!)

## Auth
- PINs als **SHA-256** gespeichert (Web Crypto API, client-side)
- Admin `4990` → `3595dba096286be744d12935ddc4920d51fb661331084890637f35ea388480e0`
- Test `1234` → `03ac674216f3e15c761ee1a5e255f067953623c8b388b4459e13f978d7c846f4`
- Test-Modus: lädt Dummy-Daten, speichert NICHT in Sheets

## Farbpalette
```js
const GOLD  = '#c9a96e'  // warm sand/beige
const SAND  = '#d4b483'  // heller Sand (Buttons, aktive Chips)
const BROWN = '#2a1608'  // sehr dunkles Braun (Button-Text)
// Body: #1e120a, Modal: #251508, Karten: rgba(61,32,16,0.2)
```

## Datenstrukturen

### Kaffee
```js
{ id, name, subtitle, roest, typ, aromen:[...],
  intensitaet, bild, laender:[], outOfStock:false,
  params:{ mahlgrad, dosis, temp, zeit, ratio } }
// mahlgrad: gespeichert 1-10, angezeigt ×3 = /30
// ratio "1:2,25" → App zeigt ≈ output_g automatisch
```

### Tee
```js
{ id, name, subtitle, sorte, aromen:[...],
  intensitaet, bild, laender:[], outOfStock:false,
  moeglicheMischungen:[{name,id}],  // Optional! Default: [{name:"pur",id:"m0"}]
  params:{ menge, temp, ziehzeit } }
```

### Bestellung
```js
{ gast, getraenk, typ, zubereitung, uhrzeit }
// zubereitung: Espresso/Cappuccino/... für Kaffee, Mischungsname für Tee, null für pur
```

## Google Sheets Aktionen (doPost aktion-Param)
`kaffees_speichern` · `tees_speichern` · `bestellung_hinzufuegen` · `bestellungen_leeren` · `tassen_speichern` · `kommentar_hinzufuegen` · `kommentar_loeschen`

## Weltkarte (nachhaltige Koordinaten-Lösung)
```js
// Koordinaten als Lat/Lon gespeichert, Pixel bei Render berechnet:
const LAENDER_LATLON = { "Indien":[20.6,79.0], ... };
function latLonToXY(lat,lon,W=900,H=460){
  return {x:Math.round((lon+180)/360*W), y:Math.round((90-lat)/180*H)};
}
const LAENDER_COORDS = Object.fromEntries(
  Object.entries(LAENDER_LATLON).map(([k,[lat,lon]])=>[k,latLonToXY(lat,lon)])
);
// Map-Image: 900×460 equirectangular, weiße Outlines auf transparentem Hintergrund
// Pins: Teardrop SVG-Pfad (Google Maps Style), Sand-Gold Farbe
```

## Bestellfluss
**Kaffee:** Karte → BestellModal → [Schritt 1: Zubereitungsart wählen] → [Schritt 2: Name (optional)] → Bestellen  
**Tee ohne Mischungen:** Karte → BestellModal → [Name (optional)] → Bestellen  
**Tee mit Mischungen:** Karte → BestellModal → [Mischung wählen] → [Name (optional)] → Bestellen

## Namen-History
```js
const NAMEN_KEY = "dennis_namen_history"
ladeGespeicherteNamen()  // → Array aus localStorage
speichereNamen(name)     // fügt vorne ein, max 20, min 2 Zeichen
loescheAlleNamen()       // löscht komplett
// Admin-Tab "Namen" zeigt alle gespeicherten Namen mit Einzellöschung
```

## Wichtige Komponenten
- `App` — Hauptkomponente, alle States, alle async Funktionen
- `Karte({item, istTee, isAdmin, onLoeschen, onBestellen, onBearbeiten, onToggleStock, tassenCount, kommentare, ...})` 
- `Weltkarte({laender})` — Map mit Teardrop-Pins
- `TassenBadge({count})` — CupIcon SVG + Zahl
- `Formular({istTee, onSpeichern, onAbbrechen, editData, editModus})` — Add/Edit
- `CupIcon({size, color})` — eigenes SVG, ersetzt ☕ Emoji
- `PinModal({onErfolg})` — SHA-256 PIN-Eingabe
- `Lader` — Ladespinner

## Tab-Rendering (IIFE Pattern — NICHT ändern!)
```js
(()=>{
  if (tab === "namen") { return ...; }
  if (tab === "bestellungen") { return ...; }
  return div(null, /* kaffee/tee content */);
})()
// Warum IIFE: Verhindert, dass mehrere Tabs gleichzeitig gerendert werden
// Ternary-Operator war fehleranfällig bei Klammer-Ungleichgewichten
```

## Bekannte Fallstricke
1. **CACHE_KEY nie erhöhen** → würde alle Bilder aus Sheets verlieren
2. **kein Babel** → kein JSX, kein Arrow-Function in className, alles `h(...)`
3. **serverLaden() überschreibt komplett** (nicht mergen!) → verhindert Duplikate
4. **Modal-Komma:** bestellModal-Block braucht `,` nach letztem `)` (war Source vieler Bugs)
5. **CupIcon** muss VOR TassenBadge definiert sein (verwendet es)
6. **autoFocus auf Inputs** vermeiden → klappt Tastatur auf und verdeckt Auswahl

## Admin-Features
- Bearbeiten / Löschen / Hinzufügen von Kaffee & Tee
- **Ausverkauft-Banner** (diagonal über Karte) — als Admin drauftippen zum Reaktivieren
- **✨ Duplikate** bereinigen (Bestellungen-Tab)
- **☕ Reset** Tassen-Zähler
- **Namen-Tab** zum Verwalten gespeicherter Bestellnamen
- Echtzeit-Update alle 10s im Admin-Modus

## Tee-Mischungen einrichten
Im Admin → Bearbeiten → Feld `moeglicheMischungen` als JSON eingeben:
```json
[{"name":"Pur","id":"m0"},{"name":"Mit Minze","id":"m1"},{"name":"Mit Ingwer","id":"m2"}]
```
