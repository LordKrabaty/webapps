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
5. Nasazení: push na GitHub — vše jede přes GitHub Pages (`https://lordkrabaty.github.io/webapps/nova-app/`); úvodní rozcestník je kořenový `index.html` (přidej tam odkaz na novou app)

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
- `APP_NAME-autosave` · `APP_NAME-gist-token` · `APP_NAME-gist-id`
- `APP_NAME-autosync` · `APP_NAME-tombstones` · `APP_NAME-syncknown` (auto-sync — viz níže)

---

## Panely (hotové v šabloně, jen naplnit)

| Panel ID | Pozice | Účel |
|---|---|---|
| `panel-library` | levý (210px) | seznam položek, filtery, tlačítka |
| `panel-main` | střed (1fr) | formulář, detail položky |
| `panel-notes` | pravý (440px) | poznámky, log, doplňkové info |

Každý panel má per-panel zoom + fullscreen expand — **neimplementovat znovu**.

---

## Export / import formát

```js
{ v: 1, items: [...], archive: [...] }
```

Import podporuje i starší klíč `teas` místo `items` (backward compat).
Import **slučuje podle `id`/data** (deduplikace, LWW — viz „Auto-sync" níže), neduplikuje. *(V `_template` zatím staré chování bez merge — portovat až na pokyn.)*

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

## GitHub Gist sync — hotová infrastruktura

Data jedou v jednom **privátním GitHub gistu** (soubor `APP_NAME-sync.json`, obsah = JSON string). Při kopírování šablony stačí:

- Nahradit `APP_NAME` → klíče i jméno sync souboru jsou správné automaticky
- Upravit `payload` v `doCloudUpload()` (ruční výběr) i `buildAutoPayload()` (autosync) pokud app ukládá víc než `items + archive`
- GitHub token (scope `gist`) a Gist ID zadá uživatel přes ⚙ modal — **nikam hardcoded**. Gist ID může nechat prázdné → sync ho při prvním pushi sám založí (`POST`); neplatné/smazané Gist ID se samo zotaví (404 → nový gist)
- Auto-sync (`☁`) běží na stejné infrastruktuře — rozšíření kolekcí viz „Auto-sync" níže
- **Bezpečný bootstrap (hotový v šabloně, nerozbít):** zapnutí `☁` i nově zadané Gist ID v ⚙ NIKDY nepushují slepě — smažou session flag `APP_NAME-cloud-checked` a zavolají `checkCloudOnStartup()` (pull → merge → až pak push). Přímý `scheduleAutoPush()` by na novém zařízení přepsal gist prázdným lokálním stavem dřív, než se stihla stáhnout cloud data
- 🔗 „odkaz pro nové zařízení" (tlačítko v ⚙ modalu, hotové v šabloně): `copySyncLink()` vygeneruje URL s Gist ID v hashi (`#gid=…&tok=…&as=1`; token jen po potvrzení, `as` = stav autosyncu), `importSyncLink()` ho při startu uloží do slotů a hash hned smaže z adresy i historie (hash se na server neposílá). Běží PŘED `checkCloudOnStartup`. Při kopírování šablony zkontroluj fallback URL v `copySyncLink` (jméno složky app)
- Šablona nese jednorázovou migraci starých `APP_NAME-jsonbin-*` klíčů → `gist-*` sloty (u nově založené app ji můžeš smazat). **Pozor:** JSONBin klíč ≠ GitHub token a Bin ID ≠ Gist ID — migrace jen přesune hodnotu do nového slotu, uživatel ji musí v ⚙ ručně přepsat.

API tvar (GET `…/gists/{id}` → `files['APP_NAME-sync.json'].content`; PATCH = přepis souboru; POST = nový gist) je hotový v `gistGet` / `gistRecord` / `gistWriteInit`.

---

## Auto-sync (tichý obousměrný LWW) — jak to chci obecně

**Cíl:** přecházet mezi zařízeními a nemyslet na to. Stáhnout při startu, nahrát po editaci, žádné ruční řešení konfliktů.

**Princip:** každá synchronizovaná položka nese `mt` (čas poslední editace, `Date.now()`). Při slučování **vyhrává novější `mt`** (last-write-wins) — **po celých záznamech, ne po jednotlivých polích**.

### Strategie podle entity

| Entita | Klíč | Slučování |
|---|---|---|
| dny / plány / šablony | datum / `id` | **LWW** (novější `mt`) |
| bloky (typy) | `key` | **LWW** (novější `mt`); vestavěný `TASK` nejde smazat, jiné defaulty nejsou |
| inbox | `id` | **LWW** (novější `mt`) |
| archiv | `id` | union (jen doplnit chybějící) |
| pracovní dny | week key | union (zatím) |
| pořadí typů bloků (`blockTypeOrder`) | — | **nesynchronizuje se** (kosmetické) |

Mazání = **tombstones** `{ "<ns>:<id>": deletedAtTs }`. Konflikt edit-vs-delete řeší čas: pozdější akce vyhrává (editace po smazání = vzkříšení).

### Pasti — NUTNÉ dodržet (jinak ztráta dat / nesynchronizuje se)

1. **`mt` razítkuj JEN při skutečné změně obsahu.** Razítkovat naslepo = prázdné přeuložení přebije novější verzi z druhého zařízení. Před razítkem porovnej `JSON.stringify` starého a nového obsahu.
2. **`mt` musí být v podpisu stavu (`sigOf`).** Jinak se editace (stejné `id`/datum/`key`) nedetekuje a merge se nespustí — změna se jen nahraje, ale nestáhne. Platí i pro entity uvnitř `settings` (bloky) — bez nich v podpisu se nesynchronizují.
3. **Živý buffer (`items`) nesmí sdílet referenci s uloženým záznamem.** `applySessionData()` dělá hlubokou kopii; sdílení reference rozbije detekci změny (`mt`).
4. **Editaci aktivního záznamu flushni do úložiště i bez přepnutí.** `schedSave()` volá `autoSavePlanDay()`, aby se změna nasynchronizovala i když uživatel jen zavře appku.
5. **Po mergi obnov živý buffer aktivního záznamu** z čerstvě sloučeného stavu — jinak ho příští flush přepíše starou verzí.
6. **Merge nesmí spustit push/reconcile** — po dobu slučování `applyingRemote = true` (žádná rekurze přes obalené save-funkce).
7. **Async cloud odpověď ulož jen do okna, pro které se stahovala** (jen u app s víc okny dne / per-okno `WK()` namespace). `silentPull()` i `checkCloudOnStartup()` si vezmou `gistId` synchronně, ale `await gistGet()` trvá — když mezitím uživatel překlikne okno, `autoImportSilent()` by sloučil stažená data do globálů (`library`/`weekDays`/…) a `saveLibrary()`/`saveWeek()` je uloží do `WK()` **aktuálního** okna → data jednoho okna přepíšou druhé. Zachyť `const winAtStart = activeWin;` na začátku a před aplikací dej `if (activeWin !== winAtStart) return;`. Registr oken (`mergeWindowRegistry`) je globální → ten slučuj **před** kontrolou. Push (`doAutoPush`/`doCloudUpload`) řešit netřeba — `gistId` i `payload` bere synchronně pohromadě, takže je vždy konzistentní.

### Pořadí položek uvnitř záznamu
Pořadí (sekvence úkolů/bloků ve dni) je **součást obsahu** — přenáší se celé a atomicky, nikdy se neslučuje po položkách ani nepřerovnává. Přeuspořádání = editace → razítkne `mt` → propíše se.

### Migrace
Stará data mají `mt = 0`. LWW při `0 vs 0` **nepřepisuje** (jen doplní chybějící) a baseline se nerazítkuje → žádná ztráta při prvním spuštění. První skutečná editace nastaví `mt` a od té chvíle vyhrává novější verze.

**Stav:** základ je v `_template/index.html` (LWW pro `library`, append-only pro `archive`, tombstones, `lwwMerge`, `autoImportSilent`, `sigOf`, `☁` tlačítko, bezpečný bootstrap pull→merge→push, 🔗 sync odkaz pro nové zařízení). Rozšiřovací body jsou označené `// APP LOGIC` — při přidání kolekce uprav: `currentKeySet()`, `buildAutoPayload()`, `autoImportSilent()`, `sigOf()`, `applyTombstones()` (jen append-only), seznam obalených save-funkcí a u object-map (bloky) přidej vlastní `mergeBlocksLww`. **Plný příklad se všemi typy kolekcí:** `todo-app/index.html` (dny, plány, šablony, inbox = LWW; bloky = LWW object-mapa; archiv = union).

---

## Responsive layout (hotový v šabloně)

| Breakpoint | Layout |
|---|---|
| > 1020px | 3 sloupce `(210px / 1fr / 440px)` |
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

## Browser pro preview a testy

Při otevírání URL, spouštění preview nebo jakémkoliv příkazu vyžadujícím prohlížeč vždy použij **Chrome**, nikdy Edge ani msedge.

- PowerShell: `Start-Process "chrome" -ArgumentList "<url>"`
- CMD / batch: `start chrome <url>`
- Přímá cesta: `& "C:\Program Files\Google\Chrome\Application\chrome.exe" "<url>"`

## Prostředí a nástroje

- Node.js není nainstalován — nepouštěj syntax check přes `node`
- Prohlížeč pro headless testy: Chrome (`C:\Program Files\Google\Chrome\Application\chrome.exe`)
- Edge není funkční — nikdy nepoužívej msedge
- Smoke testy přes headless Chrome jsou volitelné — spusť je pouze pokud o to výslovně požádám
- PowerShell je dostupný, ale preferuj přímé souborové operace kde to jde


---

## Zákazy

- ❌ Hardcoded barvy (`#fff`, `#000`, `rgba(...)`) — výhradně CSS proměnné
- ❌ `alert()` / `confirm()` — používej `customAlert()` / `customConfirm()`
- ❌ Inline styly mimo nutné výjimky (`display:none`, konkrétní px rozměry)
- ❌ npm nebo build pipeline — pouze CDN nebo čisté JS
- ❌ CSS animace a přechody (e-ink displej)
- ❌ Reimplementace komponent a funkcí které jsou hotové v šabloně