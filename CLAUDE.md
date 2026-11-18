# Octavia – OBD data pro Torque Pro / Android Auto

Kontext převzatý z konverzace v claude.ai. Komunikace v češtině.

## Vůz
- Škoda Octavia 3 kombi (typ 5E), r. v. 2018 – **ne Octavia 4** (platforma MQB, ne MQB-EVO, bez ochrany SFD)
- Motor **DKZA** – 2.0 TSI, 140 kW (190 k), EA888 Gen.3B (Millerův cyklus), 1 984 cm³, benzín
- Převodovka **DSG DQ381** (7st., mokré spojky), pohon pouze přední náprava
- ŘJ motoru: 8V0907115E, Simos 18 („R4 2.0l TFSI“) – dotaz `7E0`, odpověď `7E8`
- ŘJ převodovky: 0GC300043A („GSG DQ381“) – dotaz `7E1`, odpověď `7E9`
- Export identifikace jednotek z Car Scanneru (Transfer_Dock_Text_*.txt) obsahuje long coding všech jednotek = záloha kódování. Obsahuje VIN – nesdílet veřejně.

## Nástroje
- Torque Pro (Android) – cíl: grafické budíky v Android Auto přes plugin **AA Torque** (github.com/agronick/aa-torque, APK z Releases; vyžaduje Torque Pro, v Android Auto vývojářský režim + neznámé zdroje, udělit všechna oprávnění v AA Torque)
- Car Scanner ELM OBD2 – profil **„MQB: TSI/TFSI EA888 Gen.3B, EA888 Evo4 (Gen.4) 2.0L (with Miller cycle)“**
  - nativní Android Auto jen textově (~10 hodnot/stránka)
  - nemá plugin API → data nelze předat jiné aplikaci; adaptér zvládne jen jedno spojení najednou
- Torque nemá výrobcové profily; vlastní PIDy jako CSV ve složce `/sdcard/Android/data/org.prowl.torque/files/extendedpids/` (ne `.torque/`); trip logy v `.../files/tripLogs/`
- Car Scanner záznamy (`.brc`, jen hodnoty, ne surová komunikace) v `/sdcard/Android/data/com.ovz.carscanner/files/records/`
- **Log komunikace Car Scanneru:** Nastavení → Adaptér OBDII ELM327 → (dole) **Exportovat protokol** → sdílet `log.txt` (kumulativní od 2024, obsahuje VIN). Kopie z 2026-11-07: `work/logs/carscanner-log-2026-11-07.txt`. Log obsahuje i starší kódování/mazání DTC (`2E`, `11`, `14`, `10`) – nic z toho nepřebírat.
- ADB přes Wi-Fi: IP a port telefonu viz Bezdrátové ladění (port se mění; konkrétní hodnoty v `CLAUDE.local.md`), `adb pair` + `adb connect` mimo sandbox; před prací `ANDROID_SERIAL`. Při ovládání telefonu přes ADB uživatel nesmí telefon používat.

## Zjištěné DID (z logu, 2026-11-07)
- **Kvalt (ECM):** hl. `7E0`, `2214CB`, odpověď `62 14 CB AA`, rovnice `SIGNED(A)`: R=−1, P/N=0, 1–7 (ověřeno při jízdě 04.11. i řazení na místě 06.11.). = `[ECM] Gear` 13965471.
- **TCM gear raw:** hl. `7E1`, `223816`, 1 bajt: P/N=0x00, stupeň 1–7 = 0x02–0x08 (stupeň + 1), R=0x0C; Car Scanner [DQ200]=A−2, [DQ500] = stupeň. Ověřeno jízdou 18.11. (záznam `.brc`, [DQ500] se shoduje s `[ECM] Gear` ve všech stupních).
- Car Scanner tyto DID čte v default session (bez `10xx`).
- Rozbor dalších DID: `work/did-analysis-oil.md` (pořadí DID v testu podporovaných senzorů = pořadí ECM senzorů v CSV). Simos18 teploty = `(A*256+B)/10-273.15`.
  - Olej motoru: 7E0 `2211BD` (ověřeno v logu 12,0/28,6/39,0 °C), alt. `2215B7`; přístroje 714/77E `22202F` `A-56`.
  - Plnicí tlak skut./pož.: 7E0 `2211F0`/`2211F1` `(A*256+B)/12.8` hPa; baro `221509` `/25.6` hPa.
  - DSG: olej 7E1 `222104` (little-endian, `SIGNED(B)*256+A`), spojky `22102A`/`2B`/`2C` (stejně); `2218E3` = 7F2231.
  - Knock #1/3/4/2 `2210E2–E5` (měřítko ?), tlak paliva NT `2211C8`/`2211CE` `/10` kPa, VT `22159F`/`2211D7` (?), venkovní teplota 714 `221014` `A/2-50`.
  - Brzdy/pedály (ECM, pořadím; ověřeno Torque logem 2026-11-09): **brzdný tlak `22105C` `/10` bar** (držení 37 bar, max 139 bar); pedál brzdy `22105D` = vždy 0 (nefunguje); posilovač brzd `221490` **`/25` hPa** (podtlak; ověřeno 2026-11-15 spárováním .brc záznamu Car Scanneru 925,84/929,92 hPa s raw 0x5A6A/0x5AD0; při brzdění klesá; v Torque nastaveno); **pedál plynu `221026` `/81.92` %** (0–8192); moment `221010` `(SIGNED(A)*256+B)/10` Nm (max 385 – nad katalogem 320, orientačně). `2211F1` pod zátěží sleduje `2211F0` (~2,0 bar abs) → je to požadovaný plnicí tlak.
  - Torque trip log zapisuje teploty ve °F a tlaky v „bar“ přepočtené (÷14,5) – při analýze přepočítat.
  - V Torque upraveny rovnice přímo (názvy zůstaly „… raw (ECM)“); stav v `torque/octavia-brzdy.csv`.
  - **Spínač brzdových světel `[TCM] Stop signal switch`: 7E1 `223800`, `A` (0/1)** – ověřeno jízdou 10.11. (4 319 cyklů, shoda s `22105C`, sepne od ~0,6–1,8 bar). `22105C` správně `(SIGNED(A)*256+B)/10` (občas −0,3 bar offset).
  - **Requested brake torque, Auxiliary system (ACC) = 7E0 `221610`** (4 B, ve stání `00000000`), NE `2215E7` – určeno z pořadí dotazů na dashboardu Car Scanneru; mapování pořadím je v té oblasti posunuté. Booster raw `2211D1`, actual vacuum `221490`, booster pressure `2211CF`.
  - ABS brzdný tlak: kandidát 713 `221810` bajty B–C `/10` bar (nízká jistota); `[TCM] Break pressure` nezjištěn. Rozbor: `work/brake-analysis/` (session.txt obsahuje VIN).
  - Car Scanner čte jen senzory na zobrazené stránce (~0,12 s/dotaz). Jízda 10.11. 07:22–07:59: 58 brzdění, všechna řidičem; ACC brzdění v logu neprokázáno (`221610` se za jízdy nečetl). PIDy: `torque/octavia-brzdy-2.csv`.
  - Oprava mapování (pořadí profilu): pos 140 Requested brake torque Aux = **`2215E7`** (4 B, ve stání nenulové 0x0E1A–0x1391), pos 141 Longitudinal acceleration = `221610` (4 B, v logu vždy 0 → nepoužitelné), pos 102 Acceleration [g] = `2212B0` (FE92 / 0000), pos 135 Longitudinal acceleration raw = `2215CD` (~0x0B0C–0x0B42). Test PIDy `torque/octavia-brzdy-3.csv` + ABS `torque/octavia-abs.csv` (713 `221810` (B*256)+C, ATCRA77D) – vše naimportováno a logováno 2026-11-12; `221610` z logu vyřazen; AA Torque dash 3 View 2 = `2215E7`.
  - **Torque custom PID po pročištění (2026-11-12, 14 ks):** Baterie nabiti, Brzdny tlak raw (ECM) `22105C`, Brzdova svetla (spinac) `223800`, Kvalt (ECM), Olej DSG, Olej motoru (ECM), Olej motoru KI, Pedal plynu raw (ECM), Plnici tlak pož./skut., Posilovac brzd raw (ECM), Spojka 1 DSG, Tocivy moment (ECM), **Zrychleni vozu raw `2212B0` /1000 m/s²**. Smazáno: 221610, 2215E7, 2215CD, ABS 221810, 22105D, Spojka 2, Kvalt TCM raw, 22381D, Baterie napětí/teplota. AA Torque dash 3 View 2 = Zrychlení; dash 2 levý = Olej KI (byl prázdný). **Úprava PIDu v Torque zruší jeho vazbu v AA Torque** (budík zůstane prázdný) → po každé úpravě PID budík znovu přiřadit. Stav 2026-11-15: dash 3 = Brzdný tlak · Plnicí tlak skut. · **Točivý moment** (místo posilovače); View: Stop, Zrychlení, Plnicí tlak pož., Kvalt.
  - Formát Car Scanner `.brc`: definice senzoru = `90 61 D5 00` + ID (uint32 LE) + názvy; vzorek = `50 43 27 01` + double (čas) + ID + double (hodnota, zobrazená v jednotkách aplikace) → spárováním s logem komunikace lze přesně odvodit rovnice. Skript `brc.py` byl ve scratchpadu Windows session (není v projektu). **Pozor:** smazání custom PID ho neodebere ze seznamu logování (Záznam a odesílání dat → Vyberte pro záznam → podržet → Vymazat) – vyčištěno 2026-11-13, logováno 16 hodnot. Jízda 13.11. 15:57 bez dat (Torque nepřipojen), 14:31 utnuto po 6 min.
  - **Výsledek jízd 12.11. 16:45 a 18:04 (Torque, 1 517 vzorků):** ✔ **`2212B0` = zrychlení vozu z ESP, `(SIGNED(A)*256+B)/1000` m/s²** (korelace 0,87 se zpomalením z OBD rychlosti; regrese 1/1136, bráno /1000), ukazuje i brzdění ACC (−1 až −3 m/s² při spínači 0). ✖ `2215E7` (800–21 000, kolísá stále, není signál brzdění ACC), ✖ `2215CD` (téměř konstantní ~2910–2943), ABS 713 `221810` B–C = tlak od pedálu (jako `22105C`, ~/100 bar, 65526 = malé záporné), na ACC nereaguje. → Brzdění ACC + brzdová světla = zrychlení ≤ −0,7 m/s² při spínači 0 a pedálu plynu 0.
  - **ACC test 12.11. (Torque logy `work/torque-log/1794483670897` aj.):** `221610` = vždy 0 i při brzdění ACC → **není** použitelný signál brzdění ACC. Brzdění ACC jde poznat nepřímo: ACC aktivní (pedál plynu 0 %, moment > 30 Nm) → zpomalení ≥ 0,7 m/s² (Speed OBD) při spínači 0 a `22105C` = 0. Příklad 12:47:05–24: 48→7 km/h, max 1,67 m/s² (→ brzdová světla svítí dle předpisu). Detekce: skript `accev.py` ze scratchpadu Windows session (není v projektu). Torque log občas zapíše jen první minuty jízdy – viz „Utínání Torque logu“ níže.
  - **Jízdy 15.11. (Torque 15:00–15:13 a 17:43–17:52, obě celé):** podle zrychlení `2212B0`: 27× brzdění řidičem, 14× brzdění ACC (zrychlení ≤ −0,7 m/s², spínač 0, pedál 0 po fázi ACC; max −1,1 až −1,7 m/s², nejdelší 39→13 km/h / 43→12 km/h), 6× nejasné krátké zpomalení. Log komunikace Car Scanneru: `work/logs/carscanner-log-2026-11-15.txt` (relace 14:57 = test manuálního režimu).
  - **Manuální režim DSG (2026-11-15):** přímý senzor polohy voliče/M není; `223816` = zařazený stupeň (00 P/N, 02–06 = 1.–5.?, 0C R). Kandidáti: `[DQ500] Driving strategy: sport factor`, `[TCM] Driving time manual/sporty/ecological`, `[TCM] Kick down` – DID z logu Car Scanneru. Stránka palubní desky Car Scanneru vytvořena přes Demo (ADB): sport factor, ECM Gear, Kick down, Driving time manual, sporty, ecological (jediná stránka 1/1). Demo nezapisuje do záznamů; záznam dat po demu zase zapnut. **Výsledek relace 15.11. 14:57 (2 min):** 7E1 `221019` sport factor `A` % = vždy 0 (i při D→S→M); `22388A` driving time manual = 1 %, `223889` sporty = 0 %, `223888` ecological = 99 % (celoživotní podíly, za jízdy se nemění); `222114` kick down `A` 0/1 – potvrzeno (1 při podřazení 4→2). **D/S/M zatím rozpoznat nejde**; případně delší jízda v S/M se sport factor + otáčky/rychlost.
  - **Jízda 18.11. 18:03–18:59 (Car Scanner, `work/records/2026-11-18 18-03-08.brc`; palubní deska: sport factor, Engine RPM, Centrifugal oil var.2, Current gear [DQ500], Kick down, ECM Gear, rychlost):**
    - `[DQ500] sport factor` (`221019`) = ukazatel dynamiky jízdy, ne režim: po kick-downu nebo prudké akceleraci 33–60 % (kroky 6,67 %), za 1–2 min zpět na 0. V S při klidné jízdě 0.
    - `[TCM] Centrifugal oil temperature` i var.2 = celou jízdu 0 → DQ381 nepodporuje.
    - S proti D je vidět jen při 70–100 km/h: S drží 5.–6. stupeň (1 750–2 350 ot/min), D jede 7. stupeň od ~72 km/h (1 200–1 640 ot/min). Při 130 km/h oba 7. stupeň ~2 200 ot/min.
    - **Rozlišení D/S/M uživatel dál neřeší (2026-11-18).** Přímý signál není, odhad z kvaltu a rychlosti stačí jen pro analýzu logu.
  - **Jízdy 18.11. (Torque):** 12:50 (44 min, 38 km) log celý. Olej DSG 51–86 °C, spojka 1 až 102 °C (o 19 °C víc než olej), olej ECM max 99 °C, plnicí tlak skut. max 1,92 bar abs, 44× brzdění řidičem, 15× zpomalení typu ACC (až −2,1 m/s²). `Olej motoru KI` (`22202F`) občas 201 °C = raw 0xFF (hodnota nedostupná) → ignorovat nebo omezit max v Torque.
  - **Utínání Torque logu:** krátké jízdy (do ~14 min) celé, delší utnuté nebo prázdné (07.11., 12.11. 13:16, 13.11., 16.11. 65 min = 2 řádky). Torque má polohu jen „v popředí“ (o polohu na pozadí nežádá). Android ji na pozadí odmítá (poslední odmítnutí 16.11. 12:09 = konec jízdy). 2026-11-18 vypnuto Torque → Záznam a odesílání dat → **Automatický záznam GPS**. Potom 12:50 celý, ale 14:24 (103 min) zapsán jen do 14:36 → příčina odstraněna jen částečně. Ostatní nastavení: interval 1 s, G-senzory zapnuté, rotace logu při připojení k ECU, Torque vyjmut z optimalizace baterie.
  - Car Scanner palubní deska: přidání = dvojklep na prázdné místo → hledání (pole nahoře, nepoužívat Zpět – vyskočí z aplikace) → výběr senzoru → první vzhled (číslo). Za běhu dema uiautomator nefunguje → jen screenshoty.
  - TCM a ABS nejde mapovat pořadím (TCM 131 senzorů / 121 DID – sdílené DID; ABS 12 blokových DID pro 37 senzorů) → `[TCM] Stop signal switch`, `[TCM] Break pressure`, `[ABS] Brake pressure sensor` jen z logu, kde je Car Scanner zobrazí.
- Torque: opravy v `torque/octavia-fix-2026-11-07.csv` (naimportovat; staré `2211BE`, `015C`, `2218E3`, `221057`, `010C0D`, `01A4` v Torque smazat ručně).

## Test Torque/AA Torque v autě (2026-11-07 12:13, motor běží, chladicí 96 °C)
- ✔ Kvalt (ECM) `2214CB` (7), Olej DSG `222104` rovnice `A` (77 °C, věrohodné), Baterie nabití `222A0C` (72 %), vestavěné Coolant/IAT/Voltage/RPM/Boost.
- ✖ `2211BE` vrací `FF F2` (není teplota oleje – DID z dieselu GTD), `015C` nepodporováno (0), `2218E3` spojka 0, `221057` plnění bez odpovědi, `N/V` (`010C0D`) nesmysl 91254 – Torque neumí vícenásobný dotaz.
- ? Timing Advance −37,5° podezřelé (při 2. testu −1,5° na volnoběhu – OK).

## Test 2 (2026-11-07 15:07, po importu `octavia-fix-2026-11-07.csv`, volnoběh, chladicí 85 °C)
- ✔ Olej ECM `2211BD` 64 °C, Olej DSG 44 °C, Spojka 1 `22102B` 44 °C (= olej DSG ve stání; ověřit za jízdy), plnicí tlak skut. `2211F0` 0,98 bar abs.
- Olej KI `22202F`: s `A-56` o ~2 °C méně než zobrazuje palubní počítač → **`A-54`** (upraveno v CSV 2026-11-07; v Torque změnit ručně). Proti ECM `2211BD` se liší (jiné místo měření / filtrace).
- ✔ Turbo Boost (vestavěný): při zrychlení kladný, při plachtění záporný – správně.
- ⚠ `2211F1` = 0,35 bar abs na volnoběhu → spíš požadovaný tlak v sání, ne požadovaný plnicí tlak; ověřit pod zátěží.

## Zvolený postup (varianta 3)
Car Scanner jako průzkumník → odvodit příkaz/hlavičku/vzorec → vlastní PIDy v Torque → zobrazení přes AA Torque.

1. V Car Scanneru dashboard s vybranými hodnotami (5–8), zapnout **log komunikace** + **záznam hodnot senzorů do CSV**, jet s měnícími se podmínkami.
2. Z logu: hlavička (`ATSH7E0`/`ATSH7E1`…), dotaz `22xxxx`, odpověď `62 xx xx AA BB…`. Car Scanner může slučovat více DID v jednom dotazu (`22xxxxyyyy`) → rozdělit.
3. Vzorec odvodit spárováním surových bajtů se zobrazenými hodnotami podle času (u teplot/tlaků min. 2–3 různé hodnoty).
4. Výstup: CSV pro import do Torque (pole: Name, ShortName, ModeAndPID, Equation, Min, Max, Units, Header). V Torque rovnici `A`,`B`… = datové bajty bez `62 xx xx`.
5. Ověřit v Torque proti Car Scanneru, pak přidat do AA Torque.

Pouze čtecí příkazy (mód `22`, `01`). Nikdy `2E`, `31`, `10`.

## Senzory z profilu Car Scanneru – výběr
Doporučené pro Torque / AA:
- [DASH] Teplota motorového oleje
- ~~[TCM] Centrifugal oil temperature (var 2)~~ – u DQ381 vždy 0 (ověřeno 18.11.). Jako teplotu za spojkami použít `Spojka 1 DSG` (`22102B`).
- Kvalt: **[TCM] Current gear [DQ500]** 13965478 funguje u DQ381 (ověřeno 2026-11-06, shodné s [ECM] Gear 13965471: D=1, N/P=0, R=-1; P a N nerozlišuje, senzor voliče v profilu není – zvoleno TCM DQ500); [DQ200] varianta 13965316 dává nesmysly, obecná 13965306 nepodporována. DID zjistit z logu.
- [TCM] Clutch 1/2 actual pressure / torque
- [GATE] Battery voltage, Battery charge condition
- [DASH] Teplota okolního vzduchu
- Motor (nalezeno, ID Car Scanneru): IAT 13965178, plnicí tlak skut./pož. 13965141/140, Knock retard #1–4 13965040–043, wastegate (Charge pressure actuator: specified position) 13965186, tlak paliva NT 13965120/122, VT 13965130/131, lambda 13965035

Kompletní seznam profilu „Profil (vše)“ (528 senzorů = 411 aktivních + 117 neaktivních, přepis screenshotů `all3.jpg`+`all4.jpg` 2026-11-06): `carscanner-sensors.csv` / `.json`, sloupec `active` (1/0) a `category` (aa/live/test/check/off). Starší verze jen s aktivními v `work/carscanner-sensors_411.*`. Screenshoty se stahují z telefonu přes ADB (`/sdcard/Pictures/Screenshots/`), části do `screenshots/parts/`. Doporučení: `doporuceni-nastaveni.md`. ID 13965xxx jsou interní ID Car Scanneru, ne OBD DID.

Občasná kontrola (Car Scanner): [TCM] Clutch Temperature Protection (Qty), [DASH] Oil thermal wear, [TCM] Driving time eco/sport/manual, [ABS] TPMS, [SRS] crash counters, [GATE] battery internal resistance.

Ignorovat: CVT, AT/DL501/DL382 varianty. [DQ500]/[DQ200] položky nevypínat naslepo – DQ381 je mokrá DSG příbuzná DQ500, rozhodne test podporovaných senzorů.

Kandidáti k ověření (neověřeno pro tento vůz): `015C` hl. `7E0`, `A-40` (teplota oleje, standard OBD); `222104` hl. `7E1` (teplota DSG – převzato z VW T6, jiná převodovka).

## Otevřené úkoly
- [x] Spustit v Car Scanneru „test podporovaných senzorů“ (2026-11-06, po „povolit vše“, motor běží)
- [x] Přepsat seznam „Profil (vše)“ → sloupec `active` v CSV (2026-11-06; neaktivní = 117 položek na konci seznamu)
- [x] Přepsat seznam senzorů profilu (originály v `screenshots/`, rozdělené v `screenshots/parts/`, přepis po částech v `work/`)
- [x] Záloha Car Scanneru (.cbz = ZIP, klíč→hodnota soubory + drivecyclesv2.db + coding.bak + records/*.brc s VIN): povolení/zakázání senzorů profilu se do zálohy **neukládá** (ověřeno diffem 2026-11-06, PIDOverridesData zůstává prázdné) → vypínat ručně v aplikaci. Zálohy v `backup-orig/`.
- [ ] Neaktivní kandidáti k povolení a vyzkoušení: `[TCM] Current gear` 13965306 (pro DQ381), Knock retard var.2 13965525–528, Misfires Gen.4 13965517–524, TCM Transmission Input/Output Speed 13965297/299. Test podporovaných senzorů (po „povolit vše“, motor běží) zopakován 2026-11-06 ~14:00 (`all5.jpg`, „Profil (neaktivní)“): stejných 117 neaktivních ID → výsledek stabilní, `active` = podporováno dle testu Car Scanneru. Ruční povolení je jen pokus (test je mohl vyřadit, protože jednotka na dotaz neodpověděla).
- [ ] Pořídit log komunikace + CSV hodnot z jízdy
- [ ] Vygenerovat CSV PIDů pro Torque a otestovat – první verze kandidátů `torque/octavia-dkza-pids.csv` + `torque/dashboard.md` (2026-11-06, z Golf 7 GTD, neověřeno); `[TCM] Current gear` počítat jako nedostupný
- [x] Nastavit AA Torque – 2 dashboardy (viz `torque/dashboard.md`); `Kvalt (OBD A4)` = PID `01A4` v autě **nefunguje** (2026-11-06) → nahrazeno `Kvalt (ECM)` `2214CB` (2026-11-07 naimportováno do Torque přes `torque/octavia-kvalt-add.csv`, AA Torque View 1 = Kvalt (ECM); v Torque zůstaly staré položky `Kvalt (OBD A4)` – smazat ručně; v autě ověřit)
- Car Scanner spotřeba: doplnit reálnou cenu paliva, po pár tankováních zkalibrovat
- [ ] Návod Car Scanner CZ/EN `navod/index.html` (obrázky `navod/web/*.webp` rozmazané; `navod/raw/` = originály s VIN/MAC – nikdy nepublikovat). Hotovo bez auta (2026-11-07); 2026-11-15 doplněna kapitola „Funkce s připojením“ ze snímků v režimu Demo (palubní deska, živá data, všechny senzory, DTC, zmrazená data, monitory, ECU, testy; demo VIN je smyšlené, nerozmazává se). 2026-11-16 přidána kapitola „Torque Pro a Android Auto“ (postup, tabulka ověřených PID, ACC/brzdová světla, úskalí). Zbývá: MQB senzory v palubní desce, kódování/servis, Car Scanner v Android Auto. Návod je v gitu: github.com/mival/carscanner-manual (veřejné repo, jen bezpečné soubory, viz `.gitignore`). **Zveřejněno: https://carscanner.altrix.cz/** – Dokploy app `carscanner-altrix` (projekt Spotify Playlist, appName `carscanner-altrix-j7anzv`). Od 2026-11-16 Dokploy nasazuje z GitHubu (větev `main`, build Static, publish dir `navod`). Aktualizace = push do `main`, webhook GitHubu spustí deploy (podrobnosti v `CLAUDE.local.md`).
- Telefon: při ADB práci lze dočasně `screen_off_timeout=1800000` + `svc power stayon true`; výchozí stav 30000 a stay_on 0 (vráceno 2026-11-07, ADB odpojeno).
