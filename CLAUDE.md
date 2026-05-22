# Claude Code — instrukce pro webapps monorepo

## Struktura projektu

```
C:\dev\webapps\
├── CLAUDE.md
├── README.md
├── _template\index.html   ← základ pro každou novou app
├── tea-app\index.html
├── workout-app\index.html
└── ...
```

Každá app = samostatná složka s `index.html`.
Před prací na konkrétní app si přečti její složku — může obsahovat vlastní `CLAUDE.md` s app-specifickými doplňky.

---

## Nová aplikace — postup

1. Zkopíruj `_template\index.html` do `nova-app\index.html`
2. Globálně nahraď `APP_NAME` názvem app (např. `workout`)
3. Implementuj 4 stub funkce (viz níže)
4. Naplň 3 panely obsahem
5. Netlify: nový site, Base directory = `nova-app`

---

## 4 stub funkce k implementaci

Označeny komentářem `// APP LOGIC:` v šabloně.

| Funkce | Co implementovat |
|---|---|
| `sessionSnap()` | vrací `JSON.stringify` aktuálního stavu formuláře |
| `saveSession()` | uloží stav do `SESSION_KEY` + `activeLibId` |
| `loadSession()` | načte `SESSION_KEY`, zavolá `applySessionData()`, vrací bool |
| `saveToLibrary()` | sestaví entry objekt, pushne do `library[]`, uloží |
| `updateLibraryEntry()` | najde `activeLibId` v `library[]`, aktualizuje `entry.data` |
| `renderLibrary()` | vykreslí položky do `panel-library` |

---

## localStorage — konvence

Klíče v šabloně používají prefix `APP_NAME` — při kopírování nahradit všude, jinak dochází ke kolizi mezi appkami na stejné doméně.

- `APP_NAME-v1-session` · `APP_NAME-v1-library` · `APP_NAME-v1-archive`
- `APP_NAME-fs` · `APP_NAME-dark` · `APP_NAME-panel-fs`
- `APP_NAME-autosave` · `APP_NAME-jsonbin-key` · `APP_NAME-jsonbin-id`

---

## Panely (hotové v šabloně, jen naplnit)

| Panel ID | Pozice | Účel |
|---|---|---|
| `panel-library` | levý (210px) | seznam položek, filtery, tlačítka |
| `panel-main` | střed (1fr) | formulář, detail položky |
| `panel-notes` | pravý (330px) | poznámky, log, doplňkové info |

Každý panel má per-panel zoom + fullscreen expand — **neimplementovat znovu**.

---

## Export / import formát

```js
{ v: 1, items: [...], archive: [...] }
```

Import podporuje i starší klíč `teas` místo `items` (backward compat).
Import je bez merge flow — duplicity se přidají jako nové položky.

---

## Design systém — PŘÍSNĚ DODRŽOVAT

Pouze CSS proměnné, **nikdy hardcoded barvy**:

`--w` · `--off` · `--light` · `--mid` · `--muted` · `--dark` · `--black`

- Tmavý režim: třída `html.dark`, přepínač v headeru je hotový
- Bez barevných akcentů, bez gradientů
- Bez CSS animací — výjimka: `animation: blink step-end` (pulzující tečka)
- E-ink optimalizace: vysoký kontrast, žádné přechody

### Fonty (Google Fonts CDN, hotové v šabloně)

- `DM Sans` — UI text
- `DM Mono` — čísla, timestamps, kódy
- `Cormorant Garamond` — dekorativní nadpisy (volitelné)

---

## Hotové komponenty — nevynalézej znova

### Layout
`.card` `.card-title` `.card-title-text` — sekce obsahu s ikonou
`.setup-grid` — 2-sloupcový formulářový grid uvnitř panelu
`.notes-tabs` `.notes-tab` `.notes-pane` — záložky v panelu

### Tlačítka
`.btn-sm` `.btn-lib` `.btn-add` `.btn-io`

### Formuláře
`.field` + `label` — pole s uppercase popiskem

### Knihovna
`.lib-item` `.lib-item-name` `.lib-item-del` — řádek seznamu

### Checkbox
`.check-item` `.check-box` `.check-mark` — toggle s vizuálním stavem

### Modaly
`.modal-bg` `.modal` `.modal-row` `.modal-btn` `.modal-btn.primary` `.modal-btn.danger`

---

## Hotové funkce — volat, nepřepisovat

```js
changePanelFs('panel-id', ±1)    // zoom konkrétního panelu
toggleExpand('panel-id')          // fullscreen panel (Escape zavře)
await customConfirm(msg, label)   // nikdy nativní confirm()
await customAlert(msg)            // nikdy nativní alert()
showStatusMsg(msg, ms)            // status zpráva v headeru
showSave(msg)                     // zkratka pro showStatusMsg
schedSave()                       // debounced uložení (300ms)
pushUndo()                        // volat před každou uživatelskou změnou
closeModal('modal-id')            // zavřít modal
toggleDark()                      // přepínač dark mode
```

---

## JSONBin sync — hotová infrastruktura

Při kopírování šablony stačí:

- Nahradit `APP_NAME` → klíče jsou správné automaticky
- Upravit `payload` v `doCloudUpload()` pokud app ukládá víc než `items + archive`
- API klíč a Bin ID zadá uživatel přes ⚙ modal — **nikam hardcoded**

---

## Responsive layout (hotový v šabloně)

| Breakpoint | Layout |
|---|---|
| > 1020px | 3 sloupce `(210px / 1fr / 330px)` |
| ≤ 1020px | 2 sloupce, `panel-notes` jde pod |
| ≤ 820px | 1 sloupec, vše stacked |
| ≤ 400px | `setup-grid` přejde na 1 sloupec |

Nikdy horizontální scroll. Tlačítko ☰ skryje `library-panel`.

---

## Hardware uživatele

- Windows 11 PC (práce + domov)
- Redmi Note 15 5G (Android 14)
- Onyx Boox Note Air2 Plus (e-ink, pomalý refresh)

Vše musí být použitelné na všech třech bez nutnosti zoomu nebo horizontálního scrollu.

---

## Zákazy

- ❌ Hardcoded barvy (`#fff`, `#000`, `rgba(...)`) — výhradně CSS proměnné
- ❌ `alert()` / `confirm()` — používej `customAlert()` / `customConfirm()`
- ❌ Inline styly mimo nutné výjimky (`display:none`, konkrétní px rozměry)
- ❌ npm nebo build pipeline — pouze CDN nebo čisté JS
- ❌ CSS animace a přechody (e-ink displej)
- ❌ Reimplementace komponent a funkcí které jsou hotové v šabloně