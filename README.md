# Pokemon GO - Beste Angreifer pro Typ

## Dateien

- `beste-angreifer-pro-typ.html` — HTML mit allen 18 Typ-Tabellen (Quelle für PDF)
- `beste-angreifer-pro-typ.pdf` — Fertige PDF im A4-Querformat

## Datenquelle

**Website:** https://db.pokemongohub.net/de

**Typ-Unterseiten:** `https://db.pokemongohub.net/de/pokemon-list/best-per-type/[typ]`

Verfügbare Typen:
`normal`, `fighting`, `flying`, `poison`, `ground`, `rock`, `bug`, `ghost`, `steel`, `fire`, `water`, `grass`, `electric`, `psychic`, `ice`, `dragon`, `dark`, `fairy`

## Konfiguration auf jeder Typ-Seite

Vor dem Auslesen der Tabelle muss die **Konfiguration** über der Tabelle angepasst werden:

- Alles **ausschalten**
- Nur **"Allow mixed movesets"** aktiviert lassen

## Erhebungsprozess (Schritt für Schritt)

### 1. Tabellendaten pro Typ auslesen

Jede der 18 Typ-Seiten im Browser öffnen, Konfiguration setzen, dann per JavaScript die Tabelle auslesen:

```javascript
async () => {
    await new Promise(r => setTimeout(r, 3000)); // Warten auf React-Rendering
    const rows = document.querySelectorAll('table tbody tr');
    const seen = new Set();
    const data = [];

    for (const r of rows) {
        const cells = r.querySelectorAll('td');
        if (cells.length < 4) continue;

        // Name ohne Form-Suffix für Duplikat-Erkennung
        const name = cells[1]?.textContent?.trim();
        const baseName = name.replace(/\s*\(.*?\)\s*/g, '').trim();

        if (seen.has(baseName)) continue;
        seen.add(baseName);

        if (data.length >= 20) break;

        const link = cells[1]?.querySelector('a');
        const href = link ? link.getAttribute('href') : '';

        data.push({
            rank: data.length + 1,
            name: name,
            url: href,
            fastAttack: cells[2]?.textContent?.trim(),
            chargedAttack: cells[3]?.textContent?.trim()
        });
    }

    return data;
}
```

**Wichtig:** Die Website nutzt React/Next.js. Die Daten sind nur nach Client-Side-Rendering verfügbar. Ein einfacher `fetch()` auf die Typ-Seiten liefert keine Tabellendaten — man muss die Seite im Browser laden und nach dem Rendering auslesen.

### 2. Max WP pro Pokemon ermitteln

Die Detail-URLs der Pokemon folgen dem Muster: `/de/pokemon/[nummer]` oder `/de/pokemon/[nummer]-[form]`

Max WP kann per `fetch()` von der Detail-Seite extrahiert werden (Same-Origin, kein CORS-Problem):

```javascript
async () => {
    const urls = {
        'Pokemon-Name': '/de/pokemon/123',
        // ... weitere Pokemon
    };

    const results = {};

    for (const [name, path] of Object.entries(urls)) {
        try {
            const resp = await fetch(path);
            const html = await resp.text();
            const match = html.match(/Max WP[\s\S]*?(\d{3,5})\s*WP/);
            results[name] = match ? parseInt(match[1]) : null;
        } catch(e) {
            results[name] = null;
        }
    }

    return results;
}
```

**Tipp:** In Batches von ca. 30 Pokemon fetchen, um Timeouts zu vermeiden.

### 3. PDF generieren

```bash
google-chrome --headless --print-to-pdf="/home/skasper/Schreibtisch/pgo/beste-angreifer-pro-typ.pdf" --no-margins "file:///home/skasper/Schreibtisch/pgo/beste-angreifer-pro-typ.html"
```

## Duplikat-Regel

Verschiedene Formen desselben Pokemon (z.B. "Mewtwo" und "Mewtwo (Schatten)") zählen als Duplikat. Nur die erste (= beste) Form wird aufgenommen. Erkennung über den Base-Namen ohne Klammer-Suffix:

```javascript
const baseName = name.replace(/\s*\(.*?\)\s*/g, '').trim();
```

## Bekannte Stolpersteine

- **URL-Struktur kann sich ändern:** Ursprünglich war der Pfad `/de/best-attackers/[typ]`, aktuell ist es `/de/pokemon-list/best-per-type/[typ]`. Bei 404-Fehlern die aktuelle URL-Struktur auf der Hauptseite prüfen.
- **React-Rendering:** Die Tabellen werden erst client-seitig gerendert. Nach Navigation mindestens 3 Sekunden warten.
- **Pokedex-Nummern prüfen:** Manche Pokemon-URLs stimmen nicht exakt mit dem erwarteten Pokemon überein (z.B. regionale Formen). Im Zweifel den Namen auf der Detail-Seite gegenprüfen.

## Letzte Aktualisierung

2026-02-18
