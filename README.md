# Car Scanner manual & Octavia 3 OBD PIDs

Unofficial bilingual (CZ/EN) user guide for **Car Scanner ELM OBD2** and custom **Torque Pro / AA Torque** PIDs
for a Škoda Octavia 3 (5E) 2018, 2.0 TSI DKZA (Simos 18), DSG DQ381.

Live: https://carscanner.altrix.cz/

## Contents

| Path | What |
|---|---|
| `navod/index.html` | the guide (single page, CZ/EN switch) |
| `navod/web/` | screenshots used by the guide (personal data blurred) |
| `torque/*.csv` | Torque Pro custom PID files (import via *Manage extra PIDs → Import CSV*) |
| `torque/dashboard.md` | AA Torque dashboard layout |
| `carscanner-sensors.csv/.json` | full list of the Car Scanner connection profile sensors (528) with notes |
| `doporuceni-nastaveni.md` | recommended Car Scanner sensor settings (CZ) |
| `CLAUDE.md` | project notes: verified DIDs, equations, test results |

Not in the repo (see `.gitignore`): communication logs, Car Scanner backups and recordings, raw screenshots –
they contain the VIN and adapter MAC address.

All PIDs are read-only requests (OBD mode 01 / UDS service 22). Use at your own risk.
