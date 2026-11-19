# Torque / AA Torque – doporučený dashboard (Octavia 3, DKZA, DQ381)

PID soubor: `octavia-dkza-pids.csv` → v telefonu `/sdcard/Android/data/org.prowl.torque/files/extendedpids/` (nahráno přes ADB 2026-11-06; formát podle vzoru Torque `Other.csv`), v Torque: Nastavení → Správa extra PID → (menu) Import. Soubor **neotvírat v Excelu** (z hlavičky `7E0` udělá `7.0E+00`).

Všechny vlastní PIDy jsou **kandidáti, neověřeno na tomto voze**. Zdroj: Golf 7 GTD (MQB, diesel) – github.com/SaulGoodman1337/Golf7_GTD_Torque_Custom_PIDS (ověřené tam jen `2211BE` a `222104`, zbytek „platform-candidate“). Ověřit proti Car Scanneru; co vrací `NO DATA` / nesmysl, smazat.

Pouze čtení (mód 01/22). Zdrojový soubor GTD posílá `1003` (diagnostická session) – **vynecháno** podle pravidel projektu. Pokud `2211BE` / `222104` bez session neodpoví, je to očekávatelné a hodnotu získáme z logu Car Scanneru.

## Aktuální stav (2026-11-19, přes ADB, ověřeno náhledem)
Torque má 19 vlastních PID. Záznam (log) obsahuje 19 hodnot.

| | Dashboard 1 | Dashboard 2 | Dashboard 3 | Dashboard 4 (Motor) |
|---|---|---|---|---|
| Levý | Olej motoru (ECM) | Olej motoru (ECM) | Brzdný tlak raw (ECM) | Výfukové plyny před kat. (model) |
| Prostřední | Turbo Boost & Vacuum Gauge | Engine RPM | Plnicí tlak skutečný (abs) | Palivo VT skutečný |
| Pravý | Olej DSG | Spojka 1 DSG | Točivý moment (ECM) | Plnicí tlak skutečný (abs) |
| View 1 | Kvalt (ECM) | Timing Advance | Brzdová světla (spínač) | Klepání válec 1 |
| View 2 | Engine Coolant Temperature | Plnicí tlak požadovaný (abs) | Zrychlení vozu raw | Klepání válec 2 |
| View 3 | Intake Air Temperature | Baterie nabití | Plnicí tlak požadovaný (abs) | Klepání válec 3 |
| View 4 | Voltage (Control Module) | Plnicí tlak skutečný (abs) | Kvalt (ECM) | Klepání válec 4 |

- 2026-11-19: import `octavia-motor.csv` (EGT `221421`, VT palivo `22159F`, klepání `2210E2`/`E5`/`E3`/`E4`). Interval obnovy: EGT 2 s, VT palivo a klepání 1 s.
- Záznam: Brzdný tlak, Brzdová světla, Klepání 1–4, Kvalt, Olej DSG, Olej motoru (ECM), Palivo VT, Pedál plynu, Plnicí tlak pož./skut., Speed (OBD), Spojka 1, Točivý moment, Turbo Boost, Výfukové plyny, Zrychlení (+ G-senzory, GPS vypnuto). Ze záznamu odebráno: Throttle Position(Manifold), Posilovač brzd (PID zůstal).
- 2026-11-18: PID `Olej motoru KI` (`22202F`) smazán, protože občas vrací 0xFF (201 °C).
- Níže je historie nastavení.

## Nastaveno v AA Torque (2026-11-06, přes ADB, ověřeno náhledem)
Každý dashboard = 3 budíky + 4 hodnoty (View). Přepínání dashboardů swipe vlevo/vpravo, nahoru/dolů budík ↔ graf.

| | Dashboard 1 | Dashboard 2 (test kandidátů) |
|---|---|---|
| Levý | Olej motoru (UDS) | Olej motoru (OBD 5C) |
| Prostřední | Turbo Boost & Vacuum Gauge – převod PSI→bar, rozsah −1…2 | Engine RPM, 0…7000 |
| Pravý | Olej DSG | Teplota spojky DSG |
| View 1–4 | Kvalt (ECM) `2214CB` celá čísla (od 2026-11-07; dříve nefunkční OBD A4), Coolant, Intake Air, Voltage (Control Module) | Timing Advance, Otacky na km/h, Baterie nabiti, Plnici tlak (UDS) |

Dashboard 3 (2026-11-09, „Brzdy a tlaky“): levý Brzdný tlak `22105C` (bar 0–150), prostřední Plnicí tlak skutečný `2211F0` (bar abs 0–3), pravý Posilovač brzd `221490` (raw/256); View (od 2026-11-10): **Stop** = spínač brzdových světel `223800` (7E1, 0/1, celá čísla), **ACC brzda** = brzdný moment ACC raw `221610` (int32, měřítko neznámé), Plnicí tlak požadovaný, Kvalt. Torque logování (stav 2026-11-10): Brzdny moment ACC, Brzdny tlak, Brzdova svetla, Kvalt, Olej DSG, Olej ECM, Olej KI, Pedal plynu, Plnici tlak pož./skut., Throttle Position(Manifold), Posilovac brzd, Speed (OBD), Spojka 1, Tocivy moment, Turbo Boost (+ automaticky GPS a G-senzory). Odebráno: brzdový pedál, Spojka 2 (= Spojka 1). Brzdný tlak v Torque `(SIGNED(A)*256+B)/10`. Opravený PID pedálu plynu (`221026`, jednotka %) – dřívější ADB úprava ho poškodila.

Pozn.: AA Torque zobrazuje `Kvalt (OBD A4)` i `… TCM` pod stejným názvem (stejný PID 01A4) – nejdou rozlišit. Seznam PIDů v AA Torque: `aatorque-pid-list.txt`.

## Původní návrh – budíky (8 pro AA Torque)

| # | Budík | Zdroj | Pozn. |
|---|---|---|---|
| 1 | Teplota oleje motoru | vlastní `Olej` (0x2211BE), záloha `Olej OBD` (015C) nebo `Olej KI` (přístroje 714) | použít ten, který odpovídá Car Scanneru [DASH] |
| 2 | Teplota oleje DSG | vlastní `Olej DSG` (0x222104, rovnice `A`) | u T6 bylo `A-40` – porovnat s [TCM] Centrifugal oil temp. |
| 3 | Plnicí tlak | vestavěný Torque „Turbo Boost & Vacuum“ (MAP 010B − baro 0133) | vlastní `Plneni` jen pokud vestavěný nesedí |
| 4 | Teplota chladicí kapaliny | vestavěný 0105 | |
| 5 | Teplota nasávaného vzduchu | vestavěný 010F | |
| 6 | Napětí | vestavěný 0142 (Control module voltage), nebo `Bat V` | |
| 7 | Teplota spojky DSG | vlastní `Spojka` (0x2218E3) | kandidát |
| 8 | Předstih zážehu | vestavěný 010E | náhrada za knock retard, dokud neznáme DID |

Rezerva / druhá stránka: otáčky 010C, zatížení 0104, tlak v rail 0123 (vestavěné), `Bat SOC`, `Tlak DSG`.

## Odhad kvaltu (experimentální)
- `Kvalt` / `Kvalt T` – standardní OBD PID `01A4` (Transmission Actual Gear, SAE J1979 od ~2017), dotaz na motor `7E0` i převodovku `7E1`; kvalt = horní nibble bajtu B. Vůz 2018 ho mít může, ale nemusí → buď číslo 1–7, nebo `NO DATA`.
- `N/V` – otáčky / rychlost z jednoho dotazu `010C0D` (A,B = otáčky, C = echo `0D`, D = km/h). Každý kvalt má svůj stálý poměr → po jízdě zapsat hodnoty N/V pro 1.–7. stupeň a pak udělat převodní tabulku. Při stání (D=0) zobrazí nesmysl / ∞. Pokud Torque vícenásobný dotaz nezpracuje, ukáže nesmysl i za jízdy → smazat.

## Chybí (čeká na log Car Scanneru)
- **Rychlostní stupeň z ŘJ** – v Car Scanneru funguje `[TCM] Current gear [DQ500]` (13965478) i `[ECM] Gear` (13965471), hodnoty shodné (D=1, N/P=0, R=−1; P a N nerozlišuje). Pro budík zvolen **TCM DQ500**. Do logu komunikace zařadit oba → DID + rovnice pro Torque. Obecná `[TCM] Current gear` nepodporována, `[DQ200]` dává nesmysly.
- Knock retard #1–4, wastegate, tlak paliva NT/VT – DID zjistit z logu.
