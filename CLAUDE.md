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

## Jazyk aplikací

Aplikace se píšou **primárně v angličtině** — veškerý uživatelsky viditelný text (UI, tlačítka, tooltipy, hlášky, prompty, placeholdery), vstupní zkratky i komentáře v kódu. Platí to i pro `_template`.

Výjimka: napiš v jiném jazyce jen tehdy, když to výslovně zadám.

Zpětně-kompatibilní data (migrační mapy starých hodnot, legacy regexy) smí původní jazyk obsahovat — nejsou vidět a slouží jen k načtení existujících dat.

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

## Drag & drop (přeskládání) — VÝCHOZÍ způsob

Když řeknu „přidej drag and drop" / „ať jdou … přetahovat", použij **tenhle** vzor (ne nativní HTML5 `draggable`/`dragstart` — na dotyku nefunguje, ani knihovny z CDN). Funguje na myši, dotyku i peru.

**Princip (proč je plynulý):** během tažení se **NEpřeskupuje DOM** (žádný reflow při každém pohybu). Místo toho se na cílovém řádku ukáže jen **čára-indikátor** (kam položka spadne) a samotné přeřazení datového pole proběhne **až při puštění** → následuje jediný `render`.

**Mechanika (Pointer Events):**
- Jeden delegovaný `pointerdown` listener na kontejneru (přežije `innerHTML` re-render — kontejner se nemění). Init zavolej jednou, idempotentně (`if (container._dndInit) return;`).
- Myš: drag začne až po překročení prahu pohybu (~5px). Dotyk/pero: buď **long-press hold** (~180 ms — když je tažný celý řádek), nebo **hned** (když se tahá za vyhrazený úchyt-grip).
- Řádky mají `data-id`; přeskládává se **podle id**, ne podle indexu. Před změnou `pushUndo()`, po změně `schedSave()` + `render()`.
- Během tažení: tažený řádek dostane `.dragging` (jen `opacity`), cílový `.drag-over-top`/`.drag-over-bottom` (čára přes `box-shadow`, **žádné CSS animace** kvůli e-inku).
- `srcRow.style.pointerEvents='none'` během tažení → `elementFromPoint` vidí řádek pod prstem. Globální `touchmove` s `preventDefault` (`{passive:false}`) dokud `dndActive`, aby tažení nescrollovalo stránku. Úchyt-grip má `touch-action:none`.
- `pointermove`/`pointerup`/`pointercancel` se věší na `document` (capture) při `pointerdown` a odvěšují v `end()`.

**Úchyt vs. celý řádek:** je-li řádek plný editovatelných polí (textarea/input — např. bloky ve vision-app), přidej **vyhrazený grip** (`⠿`, 6-dot SVG) a drag spouštěj jen z něj (`e.target.closest('.grip')`); jinak (řádky bez editace — todo-app) může být tažný celý řádek a `pointerdown` ignoruje start na `button, input, textarea, select, a, [contenteditable]`.

**Šipky ↑/↓ nech** jako spolehlivou alternativu (Boox e-ink: pomalý refresh, tah je nepřesný).

**Referenční implementace:** `todo-app/index.html` → `initDnd()` + `applyDndReorder()` (tažný celý řádek, long-press na dotyku). `vision-app/index.html` → `initBlockDnd()` + `reorderBlocks()` (tažení za grip, dotyk startuje hned). `_template` zatím nemá — portovat na pokyn.

---

## Víceřádková textová pole — VÝCHOZÍ způsob (auto-grow)

Víceřádkové `textarea` se **primárně dělají jako auto-rostoucí** — pole se roztáhne podle obsahu (do rozumného stropu, pak teprve scroll), **nikdy fixní výška se scrollbarem a `resize:vertical`** (na e-inku/mobilu vypadá ošklivě a špatně se čte).

**Vzor (`autoGrow(ta)` ve vision-app):** `ta.style.height='auto'` → změř `scrollHeight` → nastav výšku; nad strop (`~55 % výšky okna`, víc v rozkliknutém panelu) zapni `overflowY:auto`, jinak `hidden`.

**Pasti — NUTNÉ dodržet:**
1. **`flex:0 0 auto` na textarea**, je-li uvnitř sloupcového flex kontejneru. Globální `textarea{flex:1}` jinak nastaví `flex-basis:0` a **inline výšku z autoGrow ignoruje** → pole zkolabuje na ~2 řádky. (Přesně tahle past byla u `.vis-ta` i `.q-block textarea`.) Dál `resize:none; overflow:hidden`.
2. **`autoGrow(this)` volej z `oninput`** a navíc vždy, když se pole **zviditelní** (přepnutí záložky, expand panelu, po načtení dat) — skryté pole má `offsetParent === null` a autoGrow se na něm vyhodnotí na nulu, takže ho po zobrazení znovu „přerosti".

**Referenční implementace:** `vision-app/index.html` → `autoGrow()`, bloky (`.vis-ta`) i Strengths (`.q-block textarea`, helper `growStrengths()`). `_template` zatím nemá — portovat na pokyn.

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

### Dopředná kompatibilita (verze schématu) — POVINNÉ při změně tvaru dat

Cíl: staré zařízení (stará cache appky), zapnuté po dlouhé době, **nesmí ztratit ani přepsat** novější data v cloudu. Dvě vrstvy, hotové v `_template`, `todo-app` i `tea-app` (nová app je tedy zdědí):

1. **Preserve-unknown v normalizérech.** Každý normalizér (`normItem`, `normLaterBucket`, `normRecur`, …) **nejdřív rozprostře původní objekt** (`{ ...it, …známá pole }`), takže pole, která přidala novější verze, **přežijí round-trip** přes starší klient místo aby je strhnul. Tím jsou *aditivní* změny (přidání pole) bezztrátové napříč verzemi. **Nikdy nepiš normalizér jako čistý whitelist** (`return { id:…, text:… }`) — to je přesně past, co zahodí cizí novější pole. *(tea-app slučuje identitou `c=>c` a archiv ukládá celé objekty → preserve-unknown splňuje automaticky.)*

2. **Guard verze schématu.** Konstanta `APP_SCHEMA` (integer, per-app vlastní počítadlo). Payload nese `schema: APP_SCHEMA` (autosync `buildAutoPayload` i ruční export/upload). Při pullu (`checkCloudOnStartup` i `silentPull`) zavolej `checkCloudSchema(record)`: když `record.schema > APP_SCHEMA`, nastaví `cloudSchemaAhead = true` → `doAutoPush()` se **odmítne** (starý build nesmí přepsat novější cloud), tlačítko `☁` zšedne na `⚠`, status hláška + jednorázový `customAlert` vyzve k reloadu. Čtení/merge běží dál (bezpečné díky preserve-unknown), pozastavený je jen upload. `schema` je metadata **mimo `sigOf`** (neovlivňuje merge).

**`APP_SCHEMA` zvyš** vždy, když uděláš změnu tvaru dat, kterou by starší klient neuměl plně reprezentovat (hlavně strukturální/breaking; čistě aditivní pole zvládne preserve-unknown samo). Guard chrání přechody **od verze, kde je nasazený, dál** — prastarou cache bez guardu zpětně neochrání.

**Stav:** základ je v `_template/index.html` (LWW pro `library`, append-only pro `archive`, tombstones, `lwwMerge`, `autoImportSilent`, `sigOf`, `☁` tlačítko, bezpečný bootstrap pull→merge→push, 🔗 sync odkaz pro nové zařízení, **forward-compat guard** `APP_SCHEMA` + `checkCloudSchema` — viz výše). Rozšiřovací body jsou označené `// APP LOGIC` — při přidání kolekce uprav: `currentKeySet()`, `buildAutoPayload()`, `autoImportSilent()`, `sigOf()`, `applyTombstones()` (jen append-only), seznam obalených save-funkcí a u object-map (bloky) přidej vlastní `mergeBlocksLww`. **Plný příklad se všemi typy kolekcí:** `todo-app/index.html` (dny, plány, šablony, inbox = LWW; bloky = LWW object-mapa; archiv = union).

---

## Version history (Gist revize) — obnova

GitHub gist drží historii revizí (každý push = revize). Tlačítko „⏱ version history" (řádek `gistHistoryRow` v ⚙ modalu) otevře seznam revizí a umožní obnovu.

**UX (plná verze v `todo-app` i `tea-app`; základní prohlížení + filtr + šetrné dotahování i v `_template` a `vision-app`):**
- Seznam revizí ukazuje datum, **název zařízení** (z `payload.device`) a v todo/tea i **souhrn obsahu** (počty kolekcí, `revSummary()`) u každé revize. Metadata se dotahují **šetrně** (viz „Rate limit" níže): **2 workery**, rozestup ~150 ms mezi requesty, **cache per `version`** v `gistRevCache` (reopen/filtr už nefetchuje), a **zastav při 403/429** (`historyRateLimited`).
- **Filtr podle zařízení** (`ghDevFilter` / `ghFilterDev` + `#ghDevFilterRow`) — dropdown se objeví až při ≥2 zařízeních; vybere se zařízení a seznam se profiltruje (typicky „chci vidět verzi z druhého zařízení"). Seznam revizí (`history[]`) se tím nezkracuje — filtr jen skrývá řádky.
- **preview** = otevře revizi v import modalu jako náhled; apply = běžný import (LWW / name-dedup), nedestruktivní.
- **restore** = otevře stejný náhled se zaškrtáváním po kolekcích/položkách, ale ve **force režimu** (`importForceRestore = true`): vybrané položky **přepíšou** lokální záznam se stejným `id` i když jsou starší (skutečný rollback) a razítkují se `mt = Date.now()`, takže se vrácený stav prosadí i na ostatní zařízení přes sync.

**Proč force, ne merge:** prostý LWW merge (`autoImportSilent`) starou verzi nikdy neprosadí — novější lokální `mt` vyhraje a tombstony blokují smazané položky. „Restore" by tak byl bez efektu pro cokoliv lokálně změněného/smazaného (typicky inbox, recurrences). Force overwrite + `mt = now` je jediný způsob, jak rollback skutečně proběhne a rozšíří se.

**Pasti:**
- `importForceRestore` resetuj na `false` v `openImportModal()` (preview ho nesmí zdědit) a nastav `true` až PO jeho zavolání v `gistHistoryRestore()`.
- Po přepisu obnov živý buffer aktivního záznamu (`refreshActiveFromLibrary()` / `applySessionData()`), jinak ho příští flush přepíše starou verzí.
- Tombstony smazaných položek se po restore samy uvolní přes save-hook (`reconcileTombstones` je smaže, protože položky zase existují) → resurrection funguje.

**Mapování force-overwrite je per-app:** `todo-app` přes společný `importRecordWins()`/`importStamp()` (ve force režimu vrací vždy `true` / `now`); `tea-app` přes vlastní `doRestoreImport()` (overwrite by id — jeho běžný import jinak jede přes name-dedup + merge akce, ne LWW). `_template` má prohlížení + preview + filtr, ale **force-restore mapování zatím ne** — portovat na pokyn.

---

## Rate limit (GitHub API) — šetrnost a backoff

GitHub limit je **na celý účet/token**, sdílený všemi appkami i zařízeními. Dva druhy: **primární** 5000 req/hod (s tokenem; anonymně jen 60 → pozor, `api.github.com/rate_limit` v prohlížeči bez `Authorization` hlavičky ukazuje ten anonymní 60, ne tvůj), a **sekundární** (nárazová ochrana proti shluku zápisů) — vrací **403/429 i když primární kvóta zbývá**. Sekundární je ten, co se snadno klepne.

**Dvě obrany (hotové, nerozbít):**
1. **Autosync push backoff** (`pushCooldownUntil`): při 403/429 z pushe si appka přečte `Retry-After` / `X-RateLimit-Reset` (jinak 60 s) a **pozastaví push do té doby** — jediný naplánovaný pokus, `scheduleAutoPush()` cooldown respektuje. Úspěšný push cooldown vynuluje. **Bez tohohle appka retryem každé 4 s drží sekundární limit nahozený a sync se nikdy nechytne.**
2. **Šetrné version-history dotahování** (viz výše): 2 workery + rozestup + cache + stop na 403/429. Původních 5 paralelních bez rozestupu = burst až `HISTORY_MAX` GETů → spolehlivě klepne sekundární limit (typicky po otevření historie během čilého editování).

**Past:** nikdy nehamerij GitHub v retry-smyčce. Jakýkoli nový opakovaný fetch (pull, metadata, upload) musí na 403/429 **couvnout** (respektovat reset/Retry-After), ne zkoušet dál. Data se nikdy neztratí — limit se sám uvolní (primární do hodiny, sekundární do pár minut), jakmile appka přestane spamovat.

---

## Responsive layout (hotový v šabloně)

| Breakpoint | Layout |
|---|---|
| > 1020px | 3 sloupce `(210px / 1fr / 440px)` |
| ≤ 1020px | 2 sloupce, `panel-notes` jde pod |
| ≤ 820px | 1 sloupec, vše stacked |
| ≤ 400px | `setup-grid` přejde na 1 sloupec |

Nikdy horizontální scroll. Tlačítko ☰ skryje `library-panel`.

**Past — useknuté panely vpravo na mobilu (`min-width:0`):** panely jsou grid-items a ty mají defaultně `min-width:auto`, takže se **nezmenší pod šířku svého nejširšího nezalomitelného obsahu**. Jediný `white-space:nowrap` nadpis (např. dlouhý `card-title-text`) tak roztáhne panel nad šířku obrazovky a `overflow-x:clip` zbytek **uřízne vpravo** (na mobilu i při minimálním zoomu). **Proto musí mít `.library-panel`, `.main-panel`, `.notes-panel` (a flex-karty co je vyplňují, např. `.lib-card`) `min-width:0`** — pak se panel smrští na šířku sloupce a `card-title-text` se zkrátí tečkami (`text-overflow:ellipsis`). Stejné pravidlo platí pro každý nový flex/grid kontejner s nezalomitelným obsahem: dej dítěti `min-width:0`. (Opraveno v `_template`, `vision-app`.)

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

## Testování v prohlížeči

- NIKDY nepoužívej `Stop-Process -Force` na `chrome.exe` — zabilo by PWA aplikace
- Pro testy vždy spouštěj Chrome s odděleným profilem: `--user-data-dir="C:\Temp\claude-test-profile"`
- Pro zavření testovacího Chrome použij `taskkill` s konkrétním PID, ne název procesu

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