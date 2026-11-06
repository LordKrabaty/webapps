# dip-app — doplňky ke kořenovému CLAUDE.md

Výpočet a evidence impregnačních roztoků (RFL a pre-dipy): laboratorní předpis, parametry, varianty s poznámkami, suroviny s alternativami.

## Terminologie v UI

Roztok = **dip** (`Dips`, `+ new dip`, `dip note`…), **nikdy „solution“** — uživatel to slovo nechce. V kódu zůstávají identifikátory `solution` / `normSolution` / `loadSolution` (nejsou vidět).

## ⚠ Žádná firemní data v repu

Repo je veřejné (GitHub Pages). **Ceny, spotřeby, kódy receptur, DPU ani seznam surovin z `_inbox-webapps/` nikdy nehardcoduj do `index.html`.** Appka startuje jen s vodou (`id: 1`); suroviny a receptury se do ní dostanou importem JSON (⇩) nebo ručně a dál jedou privátním gistem.

## Datový model

| Kolekce | Klíč | Sync |
|---|---|---|
| `library` (`dip-app-v1-library`) — roztoky | `id` | LWW po **celém roztoku včetně všech variant** (`ns lib`) |
| `materials` (`dip-app-v1-materials`) — suroviny | `id` | LWW po surovině (`ns mat`) |
| `dip-app-active-var` — zobrazená varianta per roztok | — | **nesynchronizuje se** (stav zobrazení nesmí razítkovat `mt`) |
| `dip-app-lab-view` — přepisy na lab listu per roztok `{ str, batch, title, mat:{<secId>|all: text} }` | — | **nesynchronizuje se** (pracovní nastavení navážky, ne data) |

```
solution { id, name, desc, labStr, note, vars:[variant], mt }   // labStr = doporučená síla pro lab (85 = W85), null = W100
variant  { id, name, code, date, from, mature, prep, note, secs:[section], meas:{phRF, ph, solid} }   // hmotnosti = vždy W100; mature = min. zrání celého namíchaného roztoku; prep = poznámky k přípravě (lab list)
section  { id, kind:'rf'|'latex'|'add'|'other', kname, name, mature, note, rows:[row] }   // kname = vlastní druh (jen u 'other')   // mature = min. zrání téhle části (text, „30 min")
row      { id, mat, nm, role, conc, m }        // nm/role = snapshot pro smazanou surovinu
material { id, name, role, work, note, alts:[alt], act, mt }
alt      { id, name, producer, conc, price, cons, fF, fR, vp, note }
```

- **Role suroviny** (`water`, `naoh`, `resin`, `formaldehyde`, `latex-vp|sbr|nr`, `ammonia`, `wax`, `defoamer`, `epoxy`, `isocyanate`, `other`) řídí výpočty. Neznámá role (novější build) se počítá jako `other`, ale ukládá se beze změny.
- **Druh sekce** (`kind`) určuje součty `sum RF` / `sum L` / `sum IK` a co je „latexová část". `other` = ani RF, ani latex → počítá se jen do celku (jako aditiva). **Vlastní druhy** = `other` + `kname` („Carbon black"): v nabídce jsou všechny `kname` použité kdekoli v knihovně (`customKinds()`, nic navíc se neukládá ani nesynchronizuje), nový přes „+ new kind…" (`customPrompt`). Popisek součtu sekce = `secShort()`, celkový = `c.totLabel` („sum RFL+IK+Carbon black"). Neznámý `kind` z novějšího buildu se počítá jako `other`, uloží se beze změny.
- `row.conc` (%chem) je uložené v receptu (historický záznam); když se liší od `matWork(material)`, UI to označí (čárkovaný rámeček + „≠ material x %").
- Nová varianta = kopie aktivní se **stejnými id řádků** → `varDiff()` páruje řádky podle id. Neměnit id řádků při kopii.
- Normalizéry (`normSolution` … `normAlt`) jsou preserve-unknown (`{ ...x, …}`).

## Síla (strength, W85) — NENÍ součást varianty

Receptura varianty je **vždy plná síla W100** — tak jede provoz a z ní se počítá cena i všechny parametry. Laboratorní linka (SEDU) dává větší nános, proto se lab roztoky ředí (typicky W85). Síla je jen **parametr navážky**:

- `solution.labStr` = doporučená lab síla roztoku (pole „lab strength %" v hlavičce dipu, synchronizuje se).
- Sekce **Lab weighing** (`labHtml` / `labCalc`): síla předvyplněná z `labStr`, jde přepsat (+ dávka v g); přepsaná hodnota je jen lokální view state (`dip-app-lab-view`), nerazítkuje `mt`. Navážka = stejné pravidlo jako „set solid": receptura přepočtená na dávku, vše kromě první vody × k, první voda dorovná dávku (= k dílů W100 + (1 − k) vody).
- Parametr „solid lab W85" a „Δ vs calc." u naměřené sušiny se počítají při lab síle (sušina se měří na lab roztoku).
- Lab list je i pracovní list pro laboranta: síla + dávka, pod nimi znovu název dipu (přepsatelný), u každé části a u celého roztoku min. zrání předvyplněné z receptury (přepsatelné). Přepis = jen lokální view state (`title`, `mat`), prázdné / shodné s předvolbou = bez přepisu; přepsané pole má čárkovaný rámeček. Tabulka je kompaktní (`<table>` na šířku obsahu, dlouhé názvy se zkrátí), čísla se přepisují přes `updateLab()` bez re-renderu inputů. Na konci **preparation notes** (`variant.prep` — jak míchat, teplota…): na rozdíl od ostatních polí listu jsou to **data varianty** (synchronizují se, undo, kopírují se do nové varianty), auto-grow textarea.

## Zrání (maturing)

- `section.mature` = min. zrání části (typicky RF pryskyřice 30 min), `variant.mature` = min. zrání celého namíchaného roztoku (RFL, typicky 2 h). Text, ne číslo („30 min", „2 h", „přes noc").
- Latexová část sama obvykle nezraje — zrání patří celému RFL.
- **Migrace z poznámek:** dřív bylo zrání volným textem v `section.note`. `normVariant` u sekce **bez pole `mature`** převede poznámku, která je **jen** frází zrání (`min. 30 min zrání`, `maturing min. 2 h`, `zrání min. 24 hod`…, `parseMatureNote`), na `mature` a poznámku vyprázdní; delší poznámka zůstane beze změny. Zrání z **latexové** sekce jde do `variant.mature` (celý roztok), ne do sekce. Aditivní pole → `APP_SCHEMA` beze změny.
- **Migrace (schema 1 → 2):** dřív nesla varianta `str` a hmotnosti byly **naředěný** lab roztok. `normVariant` je převede na W100 se stejnou dávkou (vše kromě první vody ÷ k, první voda dorovná; zaokrouhlení 0,01 g), `str` zahodí a `normSolution` ho přenese do `labStr` (poslední varianta, co ho měla). Lab navážka při té síle vrátí původní hmotnosti. Běží v normalizéru → i na stažená/importovaná stará data, deterministicky. `APP_SCHEMA = 2`, aby starý build (pořád nabízí `str` na variantě) nepushoval.

## Vzorce (`calcVariant`)

- `dry = m · conc / 100` (voda 0) · `% in dip = dry / Σm` · `% in solid = dry / Σdry` · `% wet = m / Σm`
- sušina = Σ dry / Σm; bez čpavku = (Σdry − NH₃) / Σm
- RF sušina = resin + formaldehyd; L sušina = řádky v sekcích `latex` bez čpavku
- PHR = RF / L × 100 (teorie 15–20)
- n(F) = resin·fF/30 + formaldehyd/30, n(R) = resin·fR/110 (fF/fR default 12/88 % sušiny pryskyřice); F/R = n(F)/n(R) (teorie 1,2–2); F, R (mol) zobrazené na 100 g sušiny
- NaOH/RF = n(NaOH)/(n(F)+n(R)), M(NaOH) = 40 · NaOH in RF = NaOH / (RF + NaOH)
- L in solid = L / Σdry (teorie > 70) · NH₄OH in L = NH₃ / L
- Syn:NR, SBR:VP z rolí latexů; VP monomer in L = Σ(VP latex · vp %) / Σ latex (jen když mají všechny VP latexy `vp`)
- cena řádku = cena za kg při `conc` (předředění vodou z dodané koncentrace aktivní alternativy) · m / Σm
- lab síla k (W85 = 0,85): sušina lab roztoku = sušina W100 · k (viz „Síla" výše)

Ověřeno proti laboratornímu listu uživatele (sušina, F/R, NaOH/RF, NaOH in RF, NH₄OH in L, strength sedí; PHR a L in solid ±0,01 kvůli skrytým desetinným místům v listu).

## Přepočty (nástroje pod receptem)

- **scale batch** — všechny m × k, zaokrouhlení na 0,01 g, zbytek do první vody
- **set solid** — drží celkovou hmotnost: vše kromě první vody × (cílová/aktuální sušina), první voda dorovná; když by ostatní složky vážily víc než dávka → odmítne a řekne maximum
- **PHR / F/R / Syn:NR / SBR:VP při stejné sušině** (`ratioRecalc()` + `toolRatio()`, druhý řádek nástrojů; placeholder = aktuální hodnota): přesouvá sušinu mezi skupinami složek tak, že jejich součet sušiny zůstane → s dávkou drženou první vodou zůstane i sušina roztoku. Uvnitř skupiny vše × stejný faktor (vnitřní poměry zůstávají); skupina bez hmotnosti dostane vše do prvního řádku s %chem. Čpavek a ostatní beze změny.
  - PHR: celá RF část (resin + formaldehyd + **NaOH**, aby NaOH/RF zůstal) × a, latexová část (sekce `latex` bez čpavku) × b; `(RF+NaOH)·a + L·b = RF+NaOH+L`, `RF·a = PHR/100·L·b` → F/R, Syn:NR, SBR:VP beze změny.
  - F/R: formaldehyd ↔ pryskyřice při stejné RF sušině (PHR zůstává): `x = (RF/30) / (t·fR/110 + (1 − fF)/30)`, fF/fR vážené přes řádky pryskyřice; minimum = pryskyřice bez formaldehydu.
  - Syn:NR: (VP + SBR) ↔ NR při stejné sušině latexů (PHR, SBR:VP zůstávají); SBR:VP: SBR ↔ VP. Zadání `70:30`, `70/30` nebo `70` (podíl první části v %).
  - Nemožné (první voda pod nulu, chybí řádek) → `customAlert`, nic se nemění. Ověřeno simulací: cíl přesně, dávka i sušina beze změny, ostatní parametry ±0,001 (zaokrouhlení na 0,01 g).
- **přepsání podílu v řádku** (`% in dip` / `% in solid` / `% wet` jsou inputy `.rc-pct`, `shareMasses()` + `onRowPct()`): přepočte `m` té složky tak, aby podíl seděl přesně; dávka zůstává, rozdíl bere **první voda** (stejné pravidlo). `% wet` první vody = **přesun vody mezi částmi**: ostatní vodní řádky rozdíl dají / vezmou poměrně ke své hmotnosti (zaokrouhlovací zbytek do největší) → dávka i sušina zůstávají. Když ostatní voda nestačí (nebo žádná není), `customConfirm` řekne, že se změní sušina (z → na), a po potvrzení se celý roztok naředí / zahustí jako set solid (ostatní × k). Bez vody v receptuře se drží ostatní hmotnosti a mění se dávka. Aplikuje se až na Enter / opuštění pole (`onchange`), Esc vrátí; nemožné (první voda pod nulu, chybí %chem) → `customAlert`, nic se nemění. Computed buňky se přepisují přes `setCell()`, které nesahá na právě editovaný input.

**Drag & drop složek** (`initRowDnd()` na stabilním `#mainBody` + `moveRowTo()`; vzor z kořenového CLAUDE.md): tah za grip `.rc-grip` vlevo od suroviny (dotyk startuje hned), indikátor čárou, přesun dat až při puštění podle id. Jde i **mezi části** — cílové zóny jsou řádky, hlavička sekce (= na začátek části) a součtový řádek sekce (= na konec, takže i do prázdné části); mimo zóny, ale u receptury → svisle nejbližší zóna. Auto-scroll u okrajů s přepočtem cíle (`dndScroller`). Šipky ↑/↓ zůstávají.

Receptura je kompaktní: `.rc` má `width:fit-content`, sloupec názvu max 15rem (na úzkém displeji zpět na plnou šířku), výběr suroviny bez rámečku (rámeček jen při hover/focus).

## Plánováno (zatím ne)

Provozní míchání, přepis na hmotnostní zlomky základních surovin (bez ředění).
