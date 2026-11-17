# CLAUDE.md

Klaboš Doubles Trénink – webová aplikace pro trénink doublů v šipkách. UI i texty jsou česky.

## Soubory

- `index.html` – **jediný aktivní soubor aplikace** (HTML + CSS + JS v jednom, bez build kroku, bez závislostí v repu).
- `klabos.webp` – logo „Klaboš“ v záhlaví (průhledné pozadí, vložené jako `<img class="logo">` v `<h1>`). Jediný další soubor v repu.
- Repozitář: https://github.com/klabans/Training-doubles (veřejný, větev `main`). Kopie na Google Disku (`G:\Můj disk\darts`) je jen záloha, needitovat ji.

## Nasazení

- GitHub Pages z `main` / root → https://klabans.github.io/Training-doubles/. **Každý push do `main` jde rovnou živě**, takže pushovat jen otestované změny.
- Repo je veřejné: nic tajného (tokeny, hesla, privátní klíče) do něj necommitovat.

## Spuštění

Stačí otevřít `index.html` v prohlížeči (nebo jakýkoli statický server). Firebase přihlášení funguje jen z domén povolených v konzoli Firebase („Authorized domains“) – `klabans.github.io` tam musí být.

## Struktura `index.html`

1. `<style>` – základní CSS, breakpointy 768 / 900 / 1000 px.
2. První `<script>` v `<head>` – `THEMES` (a = Neon Oche; b = Terč; c = Čistý sport; d = TV turnaj, výchozí – jen nálada TV grafiky, žádná loga/kopie). Každé téma je CSS řetězec + Google Font, aplikuje `applyTheme()`; volba v `localStorage.darts_theme`.
3. Hlavní `<script>` (sekce oddělené komentáři `/* ---------- … ---------- */`):
   - **konstanty**: `PHASEDEF` (hry s id `w a b r p z c t k f`), `PRESETS`, `LADDERS`, `BOB`, `ROUTE_TAB` + `route()`/`favRoute()` (cesty na zavření).
   - **terč**: `makeBoard()` (SVG), `boardize()` nahradí `<div class="target">D16</div>` terčem se zvýrazněním.
   - **logika šipek**: `replay(darts)` a `summarize(darts)`.
   - **úložiště**: `Store` (localStorage / `claude.use('db')` / Firebase).
   - **účet**: `FIREBASE_CONFIG`, `makeFirebase()`, `Cloud`, přihlašovací formulář; bez přihlášení je aplikace „gated“.
   - **stav**: `cfg` (localStorage `darts_cfg`), `run` (běžící trénink, draft v `darts_draft_v1`).
   - **pohledy**: `renderTrainIdle`, `renderTrainActive`, `renderParty`, `renderStats`, `renderHist`, `renderStyle`; 14denní plán `P14`.
   - **velká tabule**: `sbHtml(id,R)` nad terčem běžící hry – poslední hod ✓/✗ (`lastRec`) + 2–3 klíčové hodnoty dané hry, čitelné od odhodové čáry (CSS `.sb`, velikost přes container query `cqi`). Během hry (tréninku i párty) má `body` třídu `running`: skryje se logo, kicker i účet a v záhlaví zůstane jen název hry (`#sub`), aby se hra vešla na obrazovku.
   - **render + události**: `render()` přepíše celý `#main` přes `innerHTML`; jeden delegovaný `click` handler se `switch` podle `data-act`, jeden `change` handler podle `data-*` atributů.

## Datový model (důležité)

Trénink = `{id:'s'+ts, ts, durMs, darts:[...], p14?:{id,day}}`. Ukládá se přes `recOf()` (localStorage, `claude.use('db')` i Firebase). `durMs` = čistý čas tréninku (`run.act`, staví se po 5 min bez zadání); `p14` = trénink patří do 14denního plánu `id` (ts jeho začátku), den `day`. Postup plánu se odvozuje i z uložených tréninků (`p14Done`, `p14Restore`), `cfg.p14` je jen cache.

Každý záznam v `darts` je řetězec `fáze:double:zásah[:extra…]`:

| kód | hra | formát |
|---|---|---|
| `a`, `b` | řady 20·10·5 / 16·8·4·2·1 | `a:20:1`; minutá šipka volitelně s polohou `:@x,y` (souřadnice SVG terče, poloměr drátu doublu 100, y dolů; zapisuje se ťuknutím do výřezu `missPad`). Starší záznamy mají `:l/r/o/i` – `missOff()` je přepočte přibližně. Totéž platí pro `ay`, `fa/fb`, `r`, `p`, `t` |
| `ax`, `ay` | záchrana po D5 (single / double) | `ax:1:1`, `ay:2:0` |
| `fa`, `fb` | tlakový závěr (řada a / b) | `fb:16:1` |
| `rd`, `r` | náhodná pětice (losování / šipka) | `rd:20-16-3-8-12:0`, `r:16:1` |
| `p` | jedna šipka na double | `p:20:0` |
| `c` | 121 | `c:<šipek>:<ok>[:<šipek na double>]` |
| `z` | zavírání z čísla | `z:<číslo>:<ok>:<šipek>[:<na double>]` |
| `t`, `tN` | test 30 / N šipek | `t:20:1`, `t20:16:0` |
| `k`, `kn` | Bob's 27 (`kn` = do mínusu) | `k:<double>:<zásahů 0–3>`, `k:0:0` = nová hra |

- **Všechny statistiky se vždy přepočítávají z `darts` přes `replay()`**, žádný odvozený stav se neukládá. Nové hry/metriky přidávej tak, aby šly zrekonstruovat z řetězců.
- **Formát záznamů neměnit zpětně nekompatibilně** – uložená data uživatele (Firestore i localStorage) se musí dál načíst. Nové pole přidávej na konec, nové hry novým kódem fáze.
- Kódy, které se nepočítají do úspěšnosti doublů, jsou vyloučené v `summarize()` (`k, kn, rd, c, z, ax`; Bob's 27 se přičítá zvlášť).
- `run.groups` drží velikosti dávek pro „Zpět“ (`commitMany`).

## Konvence

- Kód je hutný (minifikovaný styl, krátké názvy). Drž se stávajícího stylu a nepřeformátovávej celé soubory.
- Texty do HTML vkládej přes `esc()`, pokud nepocházejí z konstant.
- Čísla formátuj přes `fP` (procenta), `fN` (desetinná čárka), data přes `fD`/`fDL` (časová zóna Europe/Prague).
- Barvy jen přes CSS proměnné (`--s1`, `--s2`, `--tx…`, `--bd-*`, `--hl*`), aby fungovala všechna 3 témata – změny vzhledu ověř ve všech tématech a na mobilu i desktopu.
- Po každé úpravě aktualizuj `BUILD` (`'verze D. M. RRRR, HH:MM'`) – zobrazuje se v patičce a slouží k ověření, že se nasadila nová verze.

## Autodarts

- Přepínač „🎯 Autodarts“ v hlavičce běžícího tréninku (`cfg.ad`, URL `cfg.adUrl`, výchozí `ws://localhost:3180/api/events` = Board Manager na stejném PC; z https stránky funguje jen localhost).
- Stav kamery: `adRun()` z každé zprávy websocketu a z `GET /api/state` (dotaz každé 3 s, `AD.pt`) – `running===false`, `connected===false` nebo `status` obsahující stop/pause = `AD.st='stop'` („zastaveno – nepočítá“); `status` jiný než `throw` (takeout, takeoutInProgress…) = `AD.st='wait'` („vytáhni šipky“, Autodarts má oranžový terč a nepočítá). `adBody()` dá `body` třídu `ad-ok` (zelená aura kolem `.bwrap .bsvg`), `ad-wait` (oranžová) nebo `ad-bad` (červená blikající: odpojeno / zastaveno), bez překreslení obrazovky; barvy `--adok`/`--adwait`/`--adbad`.
- Websocket posílá `{type:'state',data:{status,event,throws:[…]}}`, `throws` narůstá po šipce a po vytažení je prázdné; `adState()` zpracuje jen nově přibylé šipky (počítá délku pole, duplicitní zprávy ignoruje). Výchozí počet šipek v terči po připojení načte `adBase()` z `GET /api/state`; když první zpráva websocketu přijde dřív a je to hod (`event==='throwDetected'`), počítá se z ní jen poslední šipka (jinak by se první šipka hry ztratila).
- `adThrow()` mapuje šipku na aktuální hru (`a b f r p t o k`, rozcvička ATC); zásah doublu = `multiplier===2 && segmentNumber===cíl`. Minutá šipka dostane polohu `:@x,y` (`coordsX*100`, `-coordsY*100`; 1 = vnější drát doublu, y nahoru). S Autodarts dostane polohu `:@x,y` **každá** šipka (i zásah); Bob's 27, 121 a Zavírání ukládají šipky na double jako seznam na konci záznamu: `k:<double>:<zásahů>:<list>`, `c:<šipek>:<ok>:<na double>:<list>`, `z:<číslo>:<ok>:<šipek>:<na double>:<list>`, kde list = `<double><h|m>@x,y;…` (`ptsEnc`/`ptsDec`). `summarize()` vrací `pts` (všechny šipky s polohou) a `ad` (trénink s kamerou) → sekce „Rozptyl na double“ ve Statistikách (radiálně ven/dovnitř a do stran). Automatika: losování pětice, nová hra Bob's 27 / ATC, další test, kolo s méně než 3 zachycenými šipkami se při vytažení doplní jako mimo (`adPad`). Šipky posledního kola z kamery drží `AD.show` (do první šipky dalšího kola, i po vytažení): `adMarks()` je kreslí jako značky 1–3 přes terč (`.admk`, u 121/Zavírání malý terč v panelu) – jen do vytažení šipek (`AD.old`), seznam kola zůstává a `sbHtml` je vypíše nad tabulí i s tím, jak se zapsaly. Ťuknutím na šipku `adFix()` přepne zásah ↔ mimo: u `p`, `t`, `k` přímo v záznamu, u `a b f r o` zahodí zápisy od té šipky a znovu je projde `adThrow` (vynucený výsledek `th.__h`), takže se další šipky kola přepočítají na nový cíl. Každá položka drží index a text svého zápisu (`d`, `rec`); po ručním zápisu nebo „Zpět“ se oprava odmítne. U 121 a Zavírání `adCheck()` počítá zbývající skóre (double-out, přetažení = zpět na začátek kola a zbytek kola propadá) v `run.cx` a po zavření / vyčerpání šipek sám zapíše `c:…:<ok>:<na double>` / `z:…`; `cxHtml()` ukazuje zbývá, hody po kolech a poslední pokus.

## Párty hry (záložka „Párty“)

- Hry pro 1–8 hráčů u jednoho terče, oddělené od tréninku: vlastní úložiště jen v localStorage (`darts_party_cfg` = hráči/hra/volby `pcfg`, `darts_pdraft_v1` = běžící hra `PGR`, `darts_party_v1` = odehrané hry), do Firestore ani do statistik doublů nejdou.
- Hra = `{id:'g'+ts, ts, game, opt, pl:[jména], darts:[…], end, nums?, grid?, mt?}` (`nums` = čísla hráčů v Killeru, `grid` = mřížka Tic-Tac-Toe, `mt` = zápas). **Zápas na N vítězných** (volba `pcfg.ft` u každé hry, 1 = jedna hra): každá hra zápasu se ukládá jako samostatná hra s `mt:{id:'m'+ts, ft, pl:[hráči v pořadí 1. hry]}`; skóre počítá `pgMatch(G,st)` z uložených her se stejným `mt.id` + rozehrané hry (remíza se nepočítá nikomu). Po hře „▶ N. hra zápasu“ (`pleg` → `pgFinish('leg')`, začíná další hráč v pořadí), po rozhodnutí oslava vítěze zápasu a „🔁 Nový zápas“; ve statistice karta „Zápasy“. Šipka = `S20 | D20 | T20 | SB | DB | M` (mimo) `| -` (nehozená – doplnění tahu přes „Konec tahu“ nebo vytažení po < 3 šipkách z kamery), volitelně `@x,y` (poloha z Autodarts) a `*` (ručně opraveno). Tah = vždy 3 záznamy (kromě tahu, kterým hra skončí).
- Katalog `PG` (`ki cr sh hi ac go bb cu hh fv dk tt bk cd`); hratelné mají `ok:1` a háčky `init/dart/turn/rounds/rank/cells/tile/resH/res`, volitelně `dv(d,t)` (body šipky v dané hře, výchozí `d.v`), `hl(st)` (zvýraznění cíle na terči), `pts:1` (hra na body → Ø na tah, 100+… ve statistice) a `rank` může vrátit vlastní `pl` (pořadí). Hratelné: Count Up (`cu`), Shanghai (`sh`: kolo = číslo, body jen za číslo kola, S+D+T v jednom tahu = okamžitá výhra `st.sh` s oslavou `celebrate()`, volba 7/9/20 kol), Killer (`ki`: nové hry mají pravidla Autodarts – `kiAd()`, poznají se podle `opt.hm`: životy od 0 nabíjením svého čísla (S1/D2/T3) do `opt.l` (3–6) = killer; útok killera (nebo vše při Straight Off `opt.so`) bere S1/D2/T3, Hit mode `opt.hm` n/d/t omezí segment útoku i vlastního zásahu killera; životy nikdy pod 0, zásah hráče s 0 = vyřazení; ztráta života = konec killera; vlastní zásah killera bere životy sobě bez vyřazení; `opt.fr='lv'` = v 1. kole bez útoků; `opt.r` max. kol (pak rozhodují životy); statistika zabití / získané / ztracené životy. Starší uložené hry bez `hm` mají `opt.md` (all / dbl / tpl) a hrají se původními pravidly. Čísla `G.nums` náhodně v `setup()` při `pgNew`, nebo hodem slabší rukou před hrou – `pgPicking`/`pgPick`, i z kamery; vyřazení v `st.out`/`st.el`; vyhrává poslední přeživší). Další háčky: `setup(G)`, `dsub(d)` / `tt(t)` / `ttl` (popisky šipek a tahu místo bodů), `hit(d)` (barva značky), `ev(B,A,G)` (hláška při události, např. killer; vrátí-li třetí prvek `'kill'`, ukáže se místo hlášky oslava vyřazení `celebrate(t,sub,'kill')` – padající lebky a místo nadpisu velká lebka `skullSvg()` se jménem vyřazeného na čele (4. prvek), které se během ~2 s rozpije v krvavý flek se stékajícími kapkami (`.sk-nm`/`.sk-bl`/`.sk-dr`, barva `--blood`), pod ní kdo koho zabil přes `pgShooter`), `cele(A,G)` (oslava vítěze). Stav vždy z `pgReplay(G)` (`st.p` hráč na tahu, `st.rd` kolo, `st.T` tahy, `st.t` rozehraný tah, `st.out` vyřazení, `st.over`, `st.rank`), proto oprava kterékoli šipky (`pedit` + `pgSet`) jen přepíše záznam a vše se přepočítá; šipky za koncem hry se ignorují (oprava může hru znovu otevřít).
- Statistika párty (`pgStatsHtml`, přepínač `pview` Nová hra / Statistiky): období `pper` (dny podle Europe/Prague přes `dayKey`, nebo od data `pfrom`), počítá z uložených her pořadí hráčů (výhry, remíza na 1. místě = výhra všem, Ø umístění, série výher, Ø na tah, nejlepší tah, 100+/140+/180), výhry podle her, šipky (D/T/Bull), rekordy a vzájemné zápasy. Filtr hry `pgf` zúží vše na jednu hru. Každá hra má vlastní žebříček přes háček `PG[k].lb` = `{h, key(b), row(b), note}`, kde `b = {g,w,t,pts,sc:[{s,r,pl}]}` jsou data hráče v té hře (Count Up řadí podle Ø skóre přepočteného na 8 kol); nová hra má dostat vlastní `lb`. Hráč = jméno bez mezer na okrajích a bez ohledu na velikost písmen.
- Klávesnice Single/Double/Triple (`pmul`, po šipce zpět na Single) + 1–20, 25/Bull, Mimo; „Zpět“ (`pgUndo`) vrací i doplněné `-` najednou. S Autodarts je klávesnice skrytá (ruční zápis přes `pman`) a ukáže se jen jako panel opravy (`.pged`) po ťuknutí na šipku: panel ukazuje všechny 3 šipky tahu (`psel`), po opravě `pgSet` přejde na další šipku tahu, po poslední nebo „Hotovo“ se zavře. Autodarts: `adState` při `pgOn()` (záložka Párty + běžící hra) posílá šipky do `pgAd`, vytažení do `pgTakeout`. Po 3. šipce z kamery drží `phold` dohozený tah na tabuli („Dohozeno“, výzva vytáhnout šipky), další hráč se ukáže až po vytažení (`pgTakeout`) nebo tlačítkem „Další hráč“ (`pnext`); ruční zápis hold ruší. S kamerou je obrazovka párty stabilní: terč je vidět pořád a mění se jen značky šipek (jen pro rozehraný / dohozený tah, po vytažení zmizí), výzva k vytažení a řádek předchozího tahu mají místo rezervované (skrytý `visibility:hidden` / `min-height`).

## Známé zvláštnosti

- Odpočet fází je vypnutý (`run.timed=false` v `startRun`); `updTimers`, `pause`, minuty v `PHASEDEF` jsou pozůstatek. Skutečný čas měří samostatný interval do `run.act`.
- Akce `hit/miss`, `rhit/rmiss`, `thit/tmiss`, `ayhit/aymiss` UI nepoužívá (zadává se přes `vhit/vmiss` a `dhit/dmiss`).
- Zavírání z čísla sdílí počet pokusů `cfg.cn` se hrou 121.
- Firebase `apiKey` je v kódu záměrně (veřejný klientský klíč); ochrana dat závisí na pravidlech Firestore (`users/{uid}/sessions` jen pro vlastníka).
