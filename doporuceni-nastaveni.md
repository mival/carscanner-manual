# Car Scanner – doporučené nastavení profilu (Octavia 3, DKZA, DQ381)

Zdroj: `carscanner-sensors.csv` / `.json` – 528 senzorů z profilu „MQB: TSI/TFSI EA888 Gen.3B … (Miller)“ (411 aktivních, 117 neaktivních – sloupec `active`), sloupec `category`. Tabulka níže počítá jen aktivní; z neaktivních je 43× `test`, 11× `check`, 63× `off` (Haldex 4WD, přímé TPMS, AT/DL382…).

| Kategorie | Počet | Co s tím |
|---|---|---|
| `aa` | 8 | hlavní budíky pro Torque / AA Torque |
| `live` | 29 | dashboard + logování jízdy (pro odvození PIDů) |
| `test` | 54 | DQ500/DQ200 varianty – nechat rozhodnout „test podporovaných senzorů“ |
| `check` | 131 | občasná kontrola (počítadla, adaptace, opotřebení) – jinak vypnuto |
| `off` | 189 | vypnout (CVT/AT/DL501/DL382, surové duplicity, klimatizace, TPMS historie…) |

## Proč vypínat
Všechny senzory mají `Priorita:1/1` a Car Scanner je dotazuje dokola – čím méně povolených, tím rychlejší obnova hodnot. Pro jízdu nechat povolené jen `aa` + `live` (~37), `check` zapínat jednorázově.

## Postup
1. Motor běží, auto stojí → **Proveďte test podporovaných senzorů**. Nepodporované se samy vyřadí (hlavně kategorie `test`).
2. Vypnout `off`. Pozor: tlačítko „Nastavte vše na povoleno“ vrátí vše zpět.
3. Vypnout `check` (zapnout jen při kontrole).
4. Ze dvojic ověřit, která funguje, a druhou vypnout:
   - `[TCM] Current gear [DQ200]` vs `[DQ500]` vs `[ECM] Gear`
   - `[TCM] Centrifugal oil temperature` vs `var.2`; `ATF temperature (DSG/S-Tronic)` vs `var.2`
5. Dashboard `aa` + log komunikace + CSV hodnot → jízda s měnícími se podmínkami (viz CLAUDE.md, varianta 3).

## Jádro pro Android Auto (`aa`)
| Senzor | ID |
|---|---|
| [ECM] Engine RPM | 13965047 |
| [ECM] Boost pressure: actual | 13965141 |
| [ECM] Intake air temperature | 13965178 |
| [ECM] Gear | 13965471 |
| [DASH] Teplota motorového oleje | 13965438 |
| [TCM] Centrifugal oil temperature var.2 | 13965356 |
| [GATE] Battery voltage | 13965411 |
| [ECM] Boost pressure: commanded | 13965140 |

## Dříve nedohledané motorové senzory – nalezeny
- IAT: `Intake air temperature` 13965178
- plnicí tlak skutečný/požadovaný: 13965141 / 13965140
- korekce zážehu po válcích: `Knock retard #1–#4 (var.1)` 13965040–043
- wastegate: `Charge pressure actuator: specified position` 13965186
- tlak paliva NT: 13965120 (skut.) / 13965122 (pož.), VT: 13965130 / 13965131
- lambda: `Lamba actual value` 13965035

## Poznámky
- ID (13965xxx) jsou interní ID Car Scanneru, **ne** OBD DID – pro Torque je nutné odvodit `22xxxx` z logu komunikace.
- Moduly `DQ500`/`DQ200`: DQ381 je mokrá DSG příbuzná DQ500, část může fungovat – rozhodne test, ne odhad.
- Přepis ze screenshotů ponechává překlepy aplikace (Lamba, Break pressure, speicified…), aby šlo hledat v aplikaci.
