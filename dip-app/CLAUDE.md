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

```
solution { id, name, desc, note, vars:[variant], mt }
variant  { id, name, code, date, str, from, note, secs:[section], meas:{phRF, ph, solid} }
section  { id, kind:'rf'|'latex'|'add', name, note, rows:[row] }
row      { id, mat, nm, role, conc, m }        // nm/role = snapshot pro smazanou surovinu
material { id, name, role, work, note, alts:[alt], act, mt }
alt      { id, name, producer, conc, price, cons, fF, fR, vp, note }
```

- **Role suroviny** (`water`, `naoh`, `resin`, `formaldehyde`, `latex-vp|sbr|nr`, `ammonia`, `wax`, `defoamer`, `epoxy`, `isocyanate`, `other`) řídí výpočty. Neznámá role (novější build) se počítá jako `other`, ale ukládá se beze změny.
- **Druh sekce** (`kind`) určuje součty `sum RF` / `sum L` / `sum IK` a co je „latexová část".
- `row.conc` (%chem) je uložené v receptu (historický záznam); když se liší od `matWork(material)`, UI to označí (čárkovaný rámeček + „≠ material x %").
- Nová varianta = kopie aktivní se **stejnými id řádků** → `varDiff()` páruje řádky podle id. Neměnit id řádků při kopii.
- Normalizéry (`normSolution` … `normAlt`) jsou preserve-unknown (`{ ...x, …}`).

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
- strength % (W85 = 85): sušina 100% roztoku = sušina / str; ředění `Σm` g 100% roztoku + `Σm·(100/str − 1)` g vody

Ověřeno proti laboratornímu listu uživatele (sušina, F/R, NaOH/RF, NaOH in RF, NH₄OH in L, strength sedí; PHR a L in solid ±0,01 kvůli skrytým desetinným místům v listu).

## Přepočty (nástroje pod receptem)

- **scale batch** — všechny m × k, zaokrouhlení na 0,01 g, zbytek do první vody
- **set solid** — drží celkovou hmotnost: vše kromě první vody × (cílová/aktuální sušina), první voda dorovná; když by ostatní složky vážily víc než dávka → odmítne a řekne maximum

## Plánováno (zatím ne)

Provozní míchání, přepis na hmotnostní zlomky základních surovin (bez ředění), úpravy poměrů přes suchou váhu (PHR, latexy, F/R).
