# Quake Resource Planner

ESP32 + MPU6050 is a "dumb" sensor. This server does the thinking and the website shows it.

```
ESP32 --WebSocket /esp?token=...--> server.js --WebSocket /ui--> browser (phone/laptop)
      <------- {"cmd":"alert"} -----            (OpenRouter AI called from here)
```

## Files
- `server.js`  detector (STA/LTA), PGA, intensity (MMI), AI report and allocation
- `index.html` the dashboard
- `city.json`  your city's districts and resources (edit this)
- `package.json`

## Run it locally
```
npm install
DEVICE_TOKEN=pick-a-secret OPENROUTER_API_KEY=sk-or-... node server.js
```
Open http://localhost:3000. The page says **Waiting for the ESP32 to connect**. No ESP32 yet?
Press **Start virtual sensor** (or just press **Trigger earthquake**, which starts it for you) and
play with the Custom trigger sliders: strength, duration and frequency, or one of the presets.
The fake quake goes through the same detector the real sensor will use. When the ESP32 connects,
the virtual sensor stops by itself and the page switches to live data. No API key needed either
(without one it uses simple built-in rules instead of the AI).

Tip: instead of setting variables in the terminal, make a file called `.env` next to `server.js`:
```
DEVICE_TOKEN=pick-a-secret
OPENROUTER_API_KEY=sk-or-...
```

On Windows (PowerShell): `$env:DEVICE_TOKEN="pick-a-secret"; $env:OPENROUTER_API_KEY="sk-or-..."; node server.js`

## Entering your real numbers
- **Your city data** panel: type in how much of each resource you actually have (three groups: Emergency support, Provisions, Medical) and edit your districts (population, vulnerability, a note for the AI). Add or remove districts with the buttons.
- **Real numbers from the field** panel: if you know how many people are really affected or injured in a district, enter them. They replace the rough estimate (blank = estimate), the AI is told which numbers are real, and the plan can recommend more help than the shaking alone suggests. Press **Save and update plan** to re-run the report.

## Simulated shake chart
The live chart has two tabs. **Live sensor** is the ESP32 (or virtual sensor). **Simulated shake** plays the shaking you set with the Custom trigger sliders, so you can see what the sensor would have recorded.

## If the AI says it failed
The source tag on the report says why (rate limited, empty reply, cut off, and so on). The server asks for compact JSON, retries once, keeps whatever part of a cut-off reply arrived, and fills any gap with the built-in rules, so you still get a plan. Free models are often rate limited; press **Update plan** to try again.

## Reporting specific issues
Use **Report a specific issue** to add things like "one person trapped under concrete debris" with a district,
urgency and number of people. Open issues are given to the AI (or the built-in rules) when it plans, and each gets its own
plan with a priority. After adding or resolving issues, press **Update plan with these issues** to re-run the report.
Issues are saved in `incidents.json`. Without an API key, simple keyword rules decide what each issue needs
(trapped/debris -> rescue team, ambulance, crane; fire/gas -> fire engine; water/food -> provisions; and so on).

## Resource tabs
The allocation table has three tabs:
- **Emergency support**: ambulances, rescue teams, fire engines, cranes/excavators, plus vehicles: police vehicles, evacuation buses, relief trucks, water tankers, rescue boats and helicopters.
- **Provisions**: water, food packs, shelter beds, tents, blankets and clothing sets.
- **Medical**: hospital beds, medical kits and blood units.

Enter how much of each you have in the **Your city data** panel (or `city.json`). An older `city.json` that lacks newer
resource types still works; the missing ones get default values. Rescue boats are only sent to districts whose name or
note mentions water (coast, river, harbor, flooding), so put that in the district note.

## Settings page (no code changes)
Open the **Settings** panel at the bottom of the dashboard to change, live and without a restart:
- sensor sensitivity (trigger ratio, minimum shaking in g, ignore-shorter-than seconds, with Sensitive / Normal / Strict presets)
- the AI: on/off, model, the intensity it starts at, and your OpenRouter key (with a **Test the AI** button)
- the city: resources and districts

Values are saved to `settings.json` next to `server.js` (the key is stored there too, so keep that file private).
Set `ADMIN_PASSWORD` so only you can change settings. Without it, anyone with the link can.

## Settings (environment variables)
These are only the starting values. Anything saved from the Settings panel wins.
| Name | Default | Meaning |
|---|---|---|
| `DEVICE_TOKEN` | `change-me` | Secret the ESP32 must send. Change it. |
| `ADMIN_PASSWORD` | none | If set, the Settings panel asks for it before saving. |
| `OPENROUTER_API_KEY` | none | Your OpenRouter key. Keep it only on the server. |
| `OPENROUTER_MODEL` | `openrouter/free` | Model slug. `openrouter/free` picks a free model at random. |
| `AI_MIN_MMI` | `4` | Below this intensity the AI is skipped. |
| `TRIG_RATIO` | `4` | STA/LTA ratio that counts as a trigger. |
| `MIN_STA_G` | `0.01` | Shaking weaker than this (g) never triggers. Raise it if you get false alarms. |

## What the ESP32 must do
1. Connect to `wss://YOUR-HOST/esp?token=YOUR_TOKEN` (use `ws://` and the port for a local server).
2. Sample the MPU6050 at **100 Hz**, in **g** (set range to +/-2 g, divide raw by 16384).
3. About once a second, send one text message with the last batch of samples:
   `{"s":[[0.01,-0.02,0.98],[0.01,-0.01,0.99], ... ]}` (up to 500 rows, each `[x,y,z]`).
4. Listen for `{"cmd":"alert","ms":5000}` and beep / light the LED for that many milliseconds.

## Notes
- `/healthz` returns `ok`. Handy for an uptime pinger.
- Events are saved to `events.json`. Many free hosts wipe the disk on restart, so history can reset.
- The Custom trigger is rate-limited (one every 6 s) so strangers can't burn your AI quota.
- Custom trigger fires a simulated event instantly. It needs no ESP32 and no virtual sensor, and it leaves any real sensor stream alone.
- Intensity comes from PGA using a published relation (Worden et al. 2012). Magnitude can't be
  measured by one sensor, so it isn't shown. The damage and resource numbers are a toy model.
