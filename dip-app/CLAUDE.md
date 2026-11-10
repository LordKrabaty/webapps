# dip-app — doplňky ke kořenovému CLAUDE.md

Výpočet a evidence impregnačních roztoků (RFL, pre-dipy a jiné): laboratorní předpis, parametry, varianty s poznámkami, suroviny s alternativami. Dip může obsahovat jiný dip jako složku (RFL v pre-dipu).

## Terminologie v UI

Roztok = **dip** (`Dips`, `+ new dip`, `dip note`…), **nikdy „solution“** — uživatel to slovo nechce. V kódu zůstávají identifikátory `solution` / `normSolution` / `loadSolution` (nejsou vidět).

## ⚠ Žádná firemní data v repu

Repo je veřejné (GitHub Pages). **Ceny, spotřeby, kódy receptur, DPU ani seznam surovin z `_inbox-webapps/` nikdy nehardcoduj do `index.html`.** Appka startuje jen s vodou (`id: 1`); suroviny a receptury se do ní dostanou importem JSON (⇩) nebo ručně a dál jedou **end-to-end šifrovaně** přes gist (viz „Šifrovaný sync" níže).

## Šifrovaný sync (E2E) — data jsou firemní a citlivá

V gistu je **jen šifrovaná obálka**, nikdy čitelná data. „Secret" gist není šifrovaný (kdo zná ID, přečte ho bez přihlášení) a GitHub drží všechny revize navždy → nešifrovaný push = trvalý únik.

- **Obálka** v `dip-app-sync.json`: `{ v:2, schema, enc:'AES-256-GCM', kdf:'PBKDF2-SHA256', kid, slots:{ pass, rec }, iv, ct }`. Data šifruje náhodný **datový klíč** (AES-256-GCM přes Web Crypto, žádná knihovna, nové `iv` při každém uploadu); `kid` = veřejné id toho klíče. **Hlavička (`v, schema, enc, kdf, kid, slots`) je GCM additional data** (`aadOf`) → kdo má token, nemůže ji nepozorovaně změnit (dešifrování selže). Datový klíč je v obálce **zabalený dvakrát** (slot = `{ salt, iter, iv, wk }`): klíčem z **hesla** (PBKDF2-SHA256, 600 000 iterací) a klíčem ze **záchranného klíče** (160 náhodných bitů, 100 000 iterací stačí). Název zařízení je uvnitř `ct`.
- **Každá změna hesla nebo záchranného klíče = NOVÝ datový klíč** (`rekey` → `newKeys`), nikdy jen přebalení slotu. Gist drží staré revize navždy — se stejným datovým klíčem by staré heslo / záchranný klíč ze staré revize otevřel i všechna nová data. Zařízení se starým klíčem se zamknou (`badkey`), dokud na nich nezadáš nové heslo; staré revize v historii verzí pak ukazují 🔒.
- **Na zařízení** je jen `{ dek, kid, slots }` v IndexedDB (`dip-app-crypto` / `keys` / `sync`), při selhání IndexedDB jen v paměti. `dek` je **ne-exportovatelný** — surové bajty klíče existují jen chvíli uvnitř `newKeys()` / `openSlot()`. **Heslo ani záchranný klíč se nikam neukládají**, do 🔗 odkazu nejdou.
- **Heslo:** 2×, `trim()` + `normalize('NFC')` (`normPass`, mobilní klávesnice), `passWeakness()`: min. 12 znaků, ≥ 6 různých znaků, po odečtení běžných slov / čísel (password, heslo, qwerty, dipapp, …) ≥ 8 písmen. Je to hrubé síto — doporučení v UI jsou „4 náhodná slova“.
- **Záchranný klíč** (`newRecoveryKey`): 32 znaků z abecedy bez 0/O/1/I ve skupinách po 4. Zobrazí se **jednou** (`recoveryModal`: nejde zavřít kliknutím vedle, „done“ až po „I have saved…“; 📋 copy a ✉ `mailto:` koncept bez adresáta — nic nejde přes server). Zadání je tolerantní (`normRk`: velikost písmen, mezery, pomlčky).
- **Operace s klíčem** (`setPassphrase` / `regenRecoveryKey`, drží `pushing` přes `holdSync`, gist čtou přes `fetchSealed`, které nešifrovaný obsah odmítne):
  - v gistu nic našeho → **první nastavení**: nový datový + záchranný klíč (ukáže se po zavření ⚙, `pendingRk`)
  - heslo otevře slot `pass` → **odemčení** (nové zařízení) — bez rekey
  - heslo slot neotevře, ale zařízení má stejný `kid` → **změna hesla** (po potvrzení): rekey + **nový** záchranný klíč. Je to i cesta pro **zapomenuté heslo na zařízení, kde sync ještě běží**.
  - záchranný klíč + nové heslo („forgot the passphrase?“) → otevře slot `rec` → rekey se **stejným** záchranným klíčem (zůstává platný)
  - „new recovery key“ → vyžádá **aktuální heslo** (`customPrompt` typu password) → rekey + nový záchranný klíč
  - nic nesedí → odmítne. Offline s existujícím Gist ID nic z toho **nejde** (není proti čemu ověřit).
- **Jediné místo zápisu dat:** `sealPayload()` / `sealWith()` → `gistWriteInit(token, creating, content)` bere **už zapečetěný string**. Bez klíče `sealPayload` vyhodí `e.locked = 'nokey'`.
- **Čtení:** `openContent()` (bez vedlejších efektů, i pro historii verzí) → `{ rec, locked }`. **Nešifrovaný obsah se nikdy nepoužije** (`locked: 'plain'`) — nenese žádný důkaz původu, kdokoli s tokenem by mohl podstrčit data s vysokým `mt`. `gistRecord()` hlídá `cloudLock` (`nokey` / `badkey` / `plain` / `format`) — gist, který zařízení neotevře, se **neslučuje ani nepřepisuje** (push i ruční upload stojí, ☁ ukáže 🔒, při startu jednorázový `customAlert`).
- **Read-before-overwrite:** `doAutoPush` nejdřív `pullAndMerge`, když poslední **čtení** (ne vlastní push) je starší než `PUSH_PULL_MAX_AGE` (10 s) — zastaralé zařízení tak nepřepíše gist po rekey ani nové změny z druhého zařízení. Gist bez našeho souboru (`cloudEmpty`) se smí zapsat. 403/429 při tom čtení → `pushCooldownUntil` (`rateLimitWait`).
- **Ruční upload** (`doCloudUpload`) čte gist **vždy těsně před zápisem** (`cloudWriteBlock`): jiný klíč, nešifrovaný / nečitelný obsah, novější schema nebo neúspěšné čtení (offline, 403/429, 5xx) → nic se nenahraje. Bez toho by zařízení se starým klíčem (nebo starý build), kterému selhalo startovní čtení, přepsalo gist po změně hesla. 404 a gist bez našeho souboru se zapsat smí. Když gist otevřít jde, ale drží **novější data** než zařízení (`payloadMaxTs`, stejně jako v ostatních appkách — viz kořenový CLAUDE.md), upload se **zeptá** („upload anyway") — nahrát starší data jde i záměrně. Novější verze appky / jiný klíč / nečitelný obsah otázka nejsou, tam se nenahraje nic (na rozdíl od ostatních appek, kde se na novější verzi a nečitelný obsah jen ptá).
- **🔗 odkaz** se **vždy zeptá** (`importSyncLink`), kdykoli by nastavil nebo změnil Gist ID / token — **i na zařízení bez nastaveného syncu** (cizí odkaz by upload nasměroval do cizího gistu; stačí i cizí token: PATCH do gistu, kam nesmí → 404 → nový gist v cizím účtu). Bez dotazu projde jen odkaz se stejným Gist ID a tokenem (zapne nanejvýš autosync).
- **Gist ID je jen hex** (`gistIdOk`): hlídá se v ⚙ (uložení se odmítne), v 🔗 odkazu (ignoruje se) a znovu v `gistUrl()` (vyhodí `e.badId`) — hodnota jde do URL požadavku vedle tokenu, `../…` by mířilo na jiný endpoint GitHubu. Novou cestu, kudy se Gist ID dostane do URL, veď vždy přes `gistUrl()`.
- **Žádné prázdné revize:** šifrovaný obsah je při každém uploadu jiný → `syncedSig` přeskočí push, když se `localSig()` rovná cloudu.
- **`APP_SCHEMA = 3`**, `schema` je v obálce čitelně → build bez šifrování (schema 2) nepushuje plain data přes šifrovaný gist.
- **Ztracené / ukradené zařízení:** zrušit GitHub token (github.com/settings/tokens) a na zbývajícím zařízení změnit heslo (= nový datový klíč; ztracené zařízení nová data nepřečte). Úplné vyčištění historie: smazat gist na gist.github.com a v ⚙ smazat Gist ID (vznikne nový).
- **Lokální export (⇩ download JSON) šifrovaný není** — soubor pro firemní úložiště.
- **Ověřeno** (2026-11-09) end-to-end: falešný Gist API server v Pythonu + profily headless Chrome jako zařízení; 94 kontrol (nastavení, slabá hesla, připojení telefonu, změna hesla + zamčení starých zařízení, reset záchranným klíčem, nový záchranný klíč, podvržená hlavička, podstrčený nešifrovaný gist, cizí 🔗 odkaz, XSS přes import, starý build, „útočník“ s vlastní implementací formátu: staré heslo z historie nové data neotevře). **Doplněno 2026-11-10** (commit `6eac2b3`) a ověřeno zvlášť stejnou technikou (falešný Gist API server + headless Chrome, jedna stránka = jeden scénář, 86 kontrol v `dip-app`, dalších 435 v ostatních appkách a šabloně; kontrolní běh proti commitu před opravou na `_template` a `todo-app` selhal v 83 ze 175 kontrol): čtení před ručním uploadem (`cloudWriteBlock`: jiný klíč / novější schema / nešifrovaný obsah / 403 → nic se nenahraje; novější data → dotaz; smazaný gist → nový), validace Gist ID (`gistIdOk` v ⚙, v odkazu i v `gistUrl()`) a dotaz 🔗 odkazu (nové zařízení, jiný gist, jiný token, zrušení = nic se neuloží a cloud se nekontaktuje). Netestováno: víc zařízení současně a skutečné GitHub API. Při opakování: tab zavírat přes DevTools `/json/close` a až pak proces ukončit — `taskkill /F` ztratí rate-limitované zápisy localStorage (viz paměť `browser-probe-technique`).

## ⚠ Bezpečnost šifrování — POVINNĚ upozornit uživatele

Uživatel chce být **upozorněn předem**, když by nějaká změna v téhle appce mohla oslabit šifrování nebo usnadnit únik dat. Před implementací takové změny to **řekni a navrhni bezpečnější variantu** — i když o to výslovně nežádá. Týká se to hlavně:

- **cizího JS / CDN knihoven** (jakýkoli `<script src>` — i v jiné appce na `lordkrabaty.github.io`: všechny appky sdílejí origin, tj. localStorage, IndexedDB s klíčem i GitHub token)
- **vkládání dat do HTML bez `esc()`**, nových inline handlerů s daty, ID, která nejsou čísla (`numId` / `refId` v normalizérech nesmí zmizet) — XSS = token + data + možnost dešifrovat
- **posílání dat kamkoli dalšího** (jiné API, analytika, error reporting, AI služby, sdílecí odkazy s daty, data v URL, `console.log` dešifrovaných dat nebo klíčů)
- **ukládání hesla / záchranného klíče** (localStorage, IndexedDB, cookies, „zapamatovat heslo“), exportovatelného klíče, zmírnění `passWeakness` / `PASS_MIN` / iterací PBKDF2
- **přijímání nešifrovaných nebo neověřených dat z gistu**, změny obálky mimo `aadOf`, přebalení slotu bez nového datového klíče, nového zápisu do gistu mimo `sealPayload` / `sealWith`
- **nových kanálů pro data** (další soubory v gistu, jiný cloud, šifrovaný export souboru se slabším klíčem, service worker / cache s daty)
- čehokoli, co by do veřejného repa dostalo firemní data (viz výše).

Když si nejsi jistý, upozorni radši zbytečně. Drobné úpravy výpočtů / UI, které na tohle nesahají, hlásit netřeba.

## Datový model

| Kolekce | Klíč | Sync |
|---|---|---|
| `library` (`dip-app-v1-library`) — roztoky | `id` | LWW po **celém roztoku včetně všech variant** (`ns lib`) |
| `materials` (`dip-app-v1-materials`) — suroviny | `id` | LWW po surovině (`ns mat`) |
| `dip-app-lib-order` — pořadí dipů v knihovně `{ list:[id], mt }` | — | **LWW celého seznamu** (jeden `mt`, bez tombstonů; v payloadu `libOrder` až po prvním přetažení, `mt > 0`; `mt` je v `sigOf`) |
| `dip-app-active-var` — zobrazená varianta per roztok | — | **nesynchronizuje se** (stav zobrazení nesmí razítkovat `mt`) |
| `dip-app-lab-view` — přepisy na lab listu per roztok `{ str, batch, title, mat:{<secId>|all: text}, nest:{<rowId>: W}, dfrom, dhave }` | — | **nesynchronizuje se** (pracovní nastavení navážky, ne data) |

```
solution { id, name, desc, type, labStr, note, vars:[variant], used?, mt }   // type = 'rfl'|'predip'|'other'; labStr = doporučená síla pro lab (85 = W85), null = W100; used = id varianty ★ in use (jen když je nastavená)
variant  { id, name, code, date, from, mature, prep, note, secs:[section], meas:{phRF, ph, solid} }   // hmotnosti = vždy W100; mature = min. zrání celého namíchaného roztoku; prep = poznámky k přípravě (lab list)
section  { id, kind:'rf'|'latex'|'add'|'other', kname, name, mature, note, rows:[row] }   // kname = vlastní druh (jen u 'other')   // mature = min. zrání téhle části (text, „30 min")
row      { id, mat, nm, role, conc, m }        // nm/role = snapshot pro smazanou surovinu
row      { id, dip, dv, nm, role:'dip', conc, m }   // vnořený dip: dip = id roztoku, dv = id jeho varianty; nm/conc = snapshot (viz níže)
material { id, name, role, work, note, alts:[alt], act, mt }
alt      { id, name, producer, card, conc, price, cons, fF, fR, vp, note }   // card = kmenová karta (číslo položky), per alternativa
```

- **Role suroviny** (`water`, `naoh`, `resin`, `formaldehyde`, `latex-vp|sbr|nr`, `ammonia`, `wax`, `defoamer`, `epoxy`, `isocyanate`, `other`) řídí výpočty. Neznámá role (novější build) se počítá jako `other`, ale ukládá se beze změny.
- **Druh sekce** (`kind`) určuje součty `sum RF` / `sum L` / `sum additives` (druh `add` = „Additives“ — IK, vosk, odpěňovač…, ne jen izokyanát) a co je „latexová část". `other` = ani RF, ani latex → počítá se jen do celku (jako aditiva). **Vlastní druhy** = `other` + `kname` („Carbon black"): v nabídce jsou všechny `kname` použité kdekoli v knihovně (`customKinds()`, nic navíc se neukládá ani nesynchronizuje), nový přes „+ new kind…" (`customPrompt`). Popisek součtu sekce = `secShort()`, celkový = `c.totLabel` („sum RFL+additives+Carbon black"). Neznámý `kind` z novějšího buildu se počítá jako `other`, uloží se beze změny.
- `row.conc` (%chem) je uložené v receptu (historický záznam); když se liší od `matWork(material)`, UI to označí (čárkovaný rámeček + „≠ material x %").
- **Hmotnosti se zobrazují na 0,1 g** (`fmtG` v poli M (g), lab navážka) — váží se na desetiny, podíly (% in dip…) stačí na 2 desetinná místa. **Data** drží, co se zadalo / spočítalo (nástroje dál zaokrouhlují na 0,01 g, aby cíl seděl přesně) → součet sloupce se může od zobrazeného součtu lišit o 0,1. Lab navážka se počítá rovnou na 0,1 g (`roundM`), aby sloupec dal přesně dávku. Migrace schema 1 → 2 zůstává na 0,01 g (deterministická napříč buildy — neměnit).

## Varianty — pořadí a ★ in use

- **Pořadí** = pořadí v `solution.vars` (součást dipu: synchronizuje se, undo). Seznam v záložce Variants i čipy nad recepturou ho ukazují **obráceně** (`varListOrder()`; nová varianta se pushne → nahoře / vlevo). Přeskládání: drag & drop za grip v hlavičce karty (`moveVarTo`) nebo šipky ↑/↓ (`moveVar`).
- **Sdílený drag & drop seznamů** `initListDnd(box, { grip, item, group?, onDrop(id, refId, after) })` — varianty (`#varsBody`) i dipy v knihovně (`#libList`); položky nesou `data-dnd="<id>"`. Stejný vzor jako složky (dotyk startuje hned z gripu, indikátor čárou, přesun až při puštění podle id, auto-scroll, nejbližší položka); puštění tahu nesmí „kliknout" na položku (capture `click` 400 ms po dropu). `movedCopy(arr, i, j, after)` = kopie s přesunutou položkou, `null` když by zůstala na místě. Složky receptury mají vlastní `initRowDnd` (zóny sekcí).

## Pořadí dipů v knihovně

- Grip `.lib-grip` vlevo u dipu; tah jen **uvnitř skupiny typu** (`data-grp`, `group` v `initListDnd`) — typ se tažením nemění.
- Uloží se celé zobrazené pořadí všech dipů (`moveDipTo` → `libOrder = { list, mt: now }`); dip, který v seznamu není (nový, z jiného zařízení), jde za ně podle názvu (`libCmp`). Nikdy nepřetahováno = abecedně jako dřív.
- **Pořadí NENÍ v záznamu dipu** — razítkovalo by `mt` dipu a LWW celého dipu by mohlo přebít skutečnou editaci z jiného zařízení. Je to samostatný záznam (LWW celého seznamu), synchronizuje se jen autosyncem (v zapečetěném payloadu jako vše ostatní); ruční upload / export JSON ho nenesou. Starý build ho při pushi vynechá → nový ho při dalším pullu/pushi vrátí (`localSig ≠ cloudSig`).
- **★ in use** (`solution.used`, `toggleUsedVar`): jedna varianta na dip, klik na označenou ji odznačí (`delete used` → stejný tvar jako dip, který ji nikdy neměl; `normSolution` pole nepřidává). Smazání varianty ★ zruší. Zobrazí se ★ v seznamu, v čipu a v knihovně (`★ V5` za názvem dipu).
- **Výchozí varianta** = `defVarOf(sol)`: ★ in use, jinak horní v seznamu (poslední v poli). Platí pro otevření dipu bez uložené zobrazené varianty (`activeVar`), připojení dipu jako složky a náhradu smazané varianty u vnořeného dipu. „Nejnovější" se už nikde neříká — po přeskládání by to nebyla pravda.

## Typ dipu (`solution.type`)

Výběr „type" v hlavičce dipu: **RFL** · **Pre-dip** · **Other** (`DIP_TYPES`, `dipType()`). Výpočet (`calcVariant`) je pro všechny stejný — typ určuje jen, **co se zobrazí**:

| | RFL | Pre-dip | Other |
|---|---|---|---|
| parametry | vše dosavadní (PHR, F/R, NaOH…, L in solid, Syn:NR…) | solid, solid lab, solid w/o NH₃, **epoxy : IK**, cena, total | solid, solid lab, solid w/o NH₃, cena, total |
| poměrové nástroje | PHR, F/R, Syn:NR, SBR:VP | epoxy:IK | žádné (scale + set solid jen) |
| druh sekce (select) | ano | skrytý (sekce = prostá část, `kind:'other'`) | skrytý |
| součty | `sum RF` / `sum RFL+additives` | `sum` / `total` | `sum` / `total` |
| měřené | pH RF, pH RFL, solid | pH, solid | pH, solid |

- Starý záznam bez `type` = `rfl` (`normSolution` doplní). Neznámý typ z novějšího buildu se počítá jako `other`, uloží se beze změny. Aditivní pole → `APP_SCHEMA` beze změny.
- Změna typu recepturu nemění — **kromě prázdného dipu** (jedna varianta, žádná hmotnost, `isBlankDip`): ten dostane startovní recepturu typu (`defaultVariant(type)`: pre-dip = voda, epoxid, izokyanát, čpavek; other = voda). ↩ undo vrátí.
- Knihovna se seskupí podle typu (`lib-cat-header`), jakmile je v ní víc než jeden.
- Skryté hodnoty (druhy sekcí, pH RF) zůstávají v datech — po návratu na RFL jsou zpět.

## Vnořený dip (dip jako složka)

V nabídce suroviny je na konci skupina **Dips** (`value="d:<id>"`): celý jiný dip jako jedna složka (typicky RFL v pre-dipu). Pod názvem se vybírá **varianta** (`row.dv`, výchozí = varianta ★ in use, jinak horní v seznamu — v okamžiku připojení; v nabídce stejné pořadí jako seznam variant, ★ označená), „open ↗" otevře vnořený dip na té variantě.

- **%chem = sušina té varianty (W100), cena = její €/kg — živě** z její receptury (`nestedCalc`), %chem není editovatelné (kurzíva). Úprava té varianty tedy změní i nadřazený dip — hlavička vnořeného dipu to říká („component of R02 (V5 ← V3)" = varianta V5 dipu R02 používá jeho V3, `dipUsage`).
- **Obsah se započítá podle rolí** úměrně použité hmotnosti (`f = m / Σm vnořeného`): čpavek (→ solid w/o NH₃), epoxid / izokyanát (→ epoxy : IK), RF, NaOH, latexy… Latexová část: v sekci `latex` se jako L počítá celý vnořený dip bez čpavku (stejné pravidlo jako pro jinou složku), jinde jen jeho vlastní L. Pozn.: Excel uživatele odečítal u „solid no NH₄OH" jen přímý čpavek → appka vyjde o pár setin níž, pokud vnořený dip čpavek obsahuje.
- Cena vnořeného dipu, který sám nemá všechny ceny → celková cena je dolní mez (`priceOk = false`).
- **Snapshot** `nm` + `conc` (sušina) se bere jen při připojení / změně varianty — **nikdy automaticky** (razítkovalo by `mt` bez editace uživatele). Použije se jen, když vnořený dip chybí (smazaný, nestažený) nebo by byl cyklický. Smazaná varianta → použije se výchozí (`defVarOf`: ★, jinak horní) + upozornění.
- **Cykly:** dip nemůže obsahovat sám sebe ani přes jiný (`dipUses`) — nabídka je vynechá, výpočet se na cyklu zastaví (`stack` v `calcVariant`).
- Poměrové nástroje vnořený dip **neškálují** (role `dip` není v žádné skupině); `% in dip / solid / wet` a set solid / scale ano (počítají s jeho sušinou). Smazání dipu, který je složkou jinde, se v potvrzení zmíní.
- Nová varianta = kopie aktivní se **stejnými id řádků** → `varDiff()` páruje řádky podle id. Neměnit id řádků při kopii.
- Normalizéry (`normSolution` … `normAlt`) jsou preserve-unknown (`{ ...x, …}`).

## Síla (strength, W85) — NENÍ součást varianty

Receptura varianty je **vždy plná síla W100** — tak jede provoz a z ní se počítá cena i všechny parametry. Laboratorní linka (SEDU) dává větší nános, proto se lab roztoky ředí (typicky W85). Síla je jen **parametr navážky**:

- `solution.labStr` = doporučená lab síla roztoku (pole „lab strength %" v hlavičce dipu, synchronizuje se).
- Sekce **Lab weighing** (`labHtml` / `labCalc`): síla předvyplněná z `labStr`, jde přepsat (+ dávka v g); přepsaná hodnota je jen lokální view state (`dip-app-lab-view`), nerazítkuje `mt`. Navážka = stejné pravidlo jako „set solid": receptura přepočtená na dávku, vše kromě první vody × k, první voda dorovná dávku (= k dílů W100 + (1 − k) vody).
- Parametr „solid lab W85" a „Δ vs calc." u naměřené sušiny se počítají při lab síle (sušina se měří na lab roztoku).
- **Vnořený dip už naředěný** (R26 W85 obsahuje R95 W85): na lab listu má řádek vnořeného dipu vlastní sílu `W` (`labNameHtml` → input `.lab-nw`), předvyplněnou z **jeho** `labStr` (`nestRec`), přepsatelnou pro navážku (100 = plná síla; lokální view state `nest:{<rowId>: W}`, jen když se liší od předvolby; `nestStr` / `nestOver`). 1 g naředěného W_s = s g plné síly + (1 − s) g vody → v `labCalc(v, str, batch, nest)` jeho hmotnost **÷ s**, voda, kterou přinese, jde z první vody (ta dál dorovná dávku). %chem v tabulce = jeho sušina × s. Sušina lab roztoku se nemění. Bez vodního řádku nebo když by první voda šla pod nulu → hláška (max. W počítá s ÷ s). Ověřeno ručním příkladem: stejná celková voda i sušina jako při W100 složce. Pole síly je kompaktní (`.lab-nw`: bez rámečku, tečkované podtržení, rámeček jen při hover/focus, čárkovaný při přepisu) — čte se jako „W85".
- **Ředění hotového dipu vodou** (blok pod tabulkou, `updateLabDil`): z W`a` (`dfrom`, výchozí 100 — pro postupné ředění např. 90) na sílu listu W`b`: pro dávku B g = `B·b/a` g W`a` + zbytek voda; k H g W`a` (`dhave`, prázdné = 1000 g) přidat `H·(a/b − 1)` g vody → `H·a/b` g W`b`. Složení vyjde stejné jako navážka z receptury (k dílů W100 + (1 − k) vody). `a ≤ b` → hláška (vodou jen ředit). Lokální view state jako zbytek listu.
- Lab list je i pracovní list pro laboranta: síla + dávka, pod nimi znovu název dipu (přepsatelný) a hned za ním síla jako součást popisku („R26 **W85**", `#labStrTag`, živě z pole strength; pole názvu je široké podle textu — `field-sizing:content`, jinak atribut `size` z `labTitleSize`), u každé části a u celého roztoku min. zrání předvyplněné z receptury (přepsatelné). Přepis = jen lokální view state (`title`, `mat`), prázdné / shodné s předvolbou = bez přepisu; přepsané pole má čárkovaný rámeček. Tabulka je kompaktní (`<table>` na šířku obsahu, dlouhé názvy se zkrátí), čísla se přepisují přes `updateLab()` bez re-renderu inputů. Na konci **preparation notes** (`variant.prep` — jak míchat, teplota…): na rozdíl od ostatních polí listu jsou to **data varianty** (synchronizují se, undo, kopírují se do nové varianty), auto-grow textarea.

## Zrání (maturing)

- `section.mature` = min. zrání části (typicky RF pryskyřice 30 min), `variant.mature` = min. zrání celého namíchaného roztoku (RFL, typicky 2 h). Text, ne číslo („30 min", „2 h", „přes noc").
- Latexová část sama obvykle nezraje — zrání patří celému RFL.
- **Migrace z poznámek:** dřív bylo zrání volným textem v `section.note`. `normVariant` u sekce **bez pole `mature`** převede poznámku, která je **jen** frází zrání (`min. 30 min zrání`, `maturing min. 2 h`, `zrání min. 24 hod`…, `parseMatureNote`), na `mature` a poznámku vyprázdní; delší poznámka zůstane beze změny. Zrání z **latexové** sekce jde do `variant.mature` (celý roztok), ne do sekce. Aditivní pole → `APP_SCHEMA` beze změny.
- **Migrace (schema 1 → 2):** dřív nesla varianta `str` a hmotnosti byly **naředěný** lab roztok. `normVariant` je převede na W100 se stejnou dávkou (vše kromě první vody ÷ k, první voda dorovná; zaokrouhlení 0,01 g), `str` zahodí a `normSolution` ho přenese do `labStr` (poslední varianta, co ho měla). Lab navážka při té síle vrátí původní hmotnosti. Běží v normalizéru → i na stažená/importovaná stará data, deterministicky. `APP_SCHEMA = 2`, aby starý build (pořád nabízí `str` na variantě) nepushoval. (Dnes `APP_SCHEMA = 3` kvůli šifrování — viz „Šifrovaný sync".)

## Vzorce (`calcVariant`)

- `dry = m · conc / 100` (voda 0) · `% in dip = dry / Σm` · `% in solid = dry / Σdry` · `% wet = m / Σm`
- sušina = Σ dry / Σm; bez čpavku = (Σdry − NH₃) / Σm
- RF sušina = resin + formaldehyd; L sušina = řádky v sekcích `latex` bez čpavku a **bez vosku** (L = kaučuk: phr = parts per hundred rubber; vosk se do L nepočítá nikdy, ani v latexové sekci — rozhodnutí uživatele 2026-11-09, odpovídá literatuře/patentům k RFL). Vnořený dip v latexové sekci = celý bez čpavku a vosku.
- **wax phr** = sušina vosku (role `wax`, kdekoli v receptuře, vč. vnořených dipů) / L × 100 — parametr „wax / L (phr)“ jen u RFL a jen když receptura vosk má; ve `varDiff` taky
- PHR = RF / L × 100 (teorie 15–20)
- n(F) = resin·fF/30 + formaldehyd/30, n(R) = resin·fR/110 (fF/fR default 12/88 % sušiny pryskyřice); F/R = n(F)/n(R) (teorie 1,2–2); F, R (mol) zobrazené na 100 g sušiny
- NaOH/RF = n(NaOH)/(n(F)+n(R)), M(NaOH) = 40 · NaOH in RF = NaOH / (RF + NaOH)
- L in solid = L / Σdry (teorie > 70) · NH₄OH in L = NH₃ / L
- Syn:NR, SBR:VP z rolí latexů; VP monomer in L = Σ(VP latex · vp %) / Σ latex (jen když mají všechny VP latexy `vp`)
- cena řádku = cena za kg při `conc` (předředění vodou z dodané koncentrace aktivní alternativy) · m / Σm
- epoxy : IK = Σ sušina epoxidů / Σ sušina izokyanátů (vč. vnořených dipů) — vzorec ručně přepočten proti pre-dip listu uživatele (epoxy : IK i sušina sedí)
- lab síla k (W85 = 0,85): sušina lab roztoku = sušina W100 · k (viz „Síla" výše)

Ověřeno proti laboratornímu listu uživatele (sušina, F/R, NaOH/RF, NaOH in RF, NH₄OH in L, strength sedí; PHR a L in solid ±0,01 kvůli skrytým desetinným místům v listu).

## Přepočty (nástroje pod receptem)

- **scale batch** — všechny m × k, zaokrouhlení na 0,01 g (zobrazení 0,1 g — viz „Datový model"), zbytek do první vody
- **set solid** — drží celkovou hmotnost: vše kromě první vody × (cílová/aktuální sušina), první voda dorovná; když by ostatní složky vážily víc než dávka → odmítne a řekne maximum
- **PHR / F/R / Syn:NR / SBR:VP při stejné sušině** (`ratioRecalc()` + `toolRatio()`, druhý řádek nástrojů; placeholder = aktuální hodnota): přesouvá sušinu mezi skupinami složek tak, že jejich součet sušiny zůstane → s dávkou drženou první vodou zůstane i sušina roztoku. Uvnitř skupiny vše × stejný faktor (vnitřní poměry zůstávají); skupina bez hmotnosti dostane vše do prvního řádku s %chem. Čpavek a ostatní beze změny.
  - PHR: celá RF část (resin + formaldehyd + **NaOH**, aby NaOH/RF zůstal) × a, latexová část (sekce `latex` bez čpavku a vosku) **+ vosk (kdekoli)** × b; `(RF+NaOH)·a + (L+W)·b = RF+NaOH+L+W`, `RF·a = PHR/100·L·b` → F/R, Syn:NR, SBR:VP i wax phr beze změny.
  - F/R: formaldehyd ↔ pryskyřice při stejné RF sušině (PHR zůstává): `x = (RF/30) / (t·fR/110 + (1 − fF)/30)`, fF/fR vážené přes řádky pryskyřice; minimum = pryskyřice bez formaldehydu.
  - Syn:NR: (VP + SBR) ↔ NR při stejné sušině latexů (PHR, SBR:VP zůstávají); SBR:VP: SBR ↔ VP. Zadání `70:30`, `70/30` nebo `70` (podíl první části v %).
  - epoxy:IK (jen pre-dip): epoxid ↔ izokyanát při stejném součtu jejich sušiny; zadání `0.2` nebo `1:5`.
  - Nemožné (první voda pod nulu, chybí řádek) → `customAlert`, nic se nemění. Ověřeno simulací: cíl přesně, dávka i sušina beze změny, ostatní parametry ±0,001 (zaokrouhlení na 0,01 g).
- **přepsání podílu v řádku** (`% in dip` / `% in solid` / `% wet` jsou inputy `.rc-pct`, `shareMasses()` + `onRowPct()`): přepočte `m` té složky tak, aby podíl seděl přesně; dávka zůstává, rozdíl bere **první voda** (stejné pravidlo). `% wet` první vody = **přesun vody mezi částmi**: ostatní vodní řádky rozdíl dají / vezmou poměrně ke své hmotnosti (zaokrouhlovací zbytek do největší) → dávka i sušina zůstávají. Když ostatní voda nestačí (nebo žádná není), `customConfirm` řekne, že se změní sušina (z → na), a po potvrzení se celý roztok naředí / zahustí jako set solid (ostatní × k). Bez vody v receptuře se drží ostatní hmotnosti a mění se dávka. Aplikuje se až na Enter / opuštění pole (`onchange`), Esc vrátí; nemožné (první voda pod nulu, chybí %chem) → `customAlert`, nic se nemění. Computed buňky se přepisují přes `setCell()`, které nesahá na právě editovaný input.

**Drag & drop složek** (`initRowDnd()` na stabilním `#mainBody` + `moveRowTo()`; vzor z kořenového CLAUDE.md): tah za grip `.rc-grip` vlevo od suroviny (dotyk startuje hned), indikátor čárou, přesun dat až při puštění podle id. Jde i **mezi části** — cílové zóny jsou řádky, hlavička sekce (= na začátek části) a součtový řádek sekce (= na konec, takže i do prázdné části); mimo zóny, ale u receptury → svisle nejbližší zóna. Auto-scroll u okrajů s přepočtem cíle (`dndScroller`). Šipky ↑/↓ zůstávají.

Receptura je kompaktní: `.rc` má `width:fit-content`, sloupec názvu max 15rem (na úzkém displeji zpět na plnou šířku), výběr suroviny bez rámečku (rámeček jen při hover/focus).

## Plánováno (zatím ne)

Provozní míchání, přepis na hmotnostní zlomky základních surovin (bez ředění).
