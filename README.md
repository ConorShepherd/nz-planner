# NZ North → South Trip Planner — Project Knowledge

> **Source of truth:** This document is reverse-engineered from `index.html` (dev branch, October 2026 — North Island + Cook Strait build).
> The March 2026 South Island trip has been folded into this build; its activities, campsites and road notes are still in the catalogue.

---

## People & Trip Context

- **Travellers:** 2 people (Conor + partner)
- **Vehicle:** campervan — configurable as self-contained or not (affects campsite options)
- **Default trip:** Fri 18 Dec 2026 → Sat 30 Jan 2027 (44 days incl. fly day)
- **Default arrival:** AKL — "Arrive Auckland — pick up campervan"
- **Default departure:** CHC — "Return campervan in Christchurch" (flight destination / time left blank)
- **Island split target:** at least 18 days (~2.5 weeks) on the North Island
- **Comfortable day:** 10–14 km / 400–600 m / 3–5 h
- **Big day:** 16–18 km / 800–1,400 m / 6–9 h (use ×0.80 on DOC times; NI walk times use ~0.75× DOC pace)
- **Hard day:** cumulative gain >800 m OR total committed hours >10 h
- **Max consecutive hard days:** 2–3 (flagged at 3+)
- **Peak season:** Kiwi summer holidays run ~26 Dec – mid Jan — ferries, Great Walk huts and popular tours sell out

---

## Repo & Deployment

- **Repo:** https://github.com/conorshepherd/nz-planner
- **Live (main):** https://conorshepherd.github.io/nz-planner/
- **Dev branch:** https://conorshepherd.github.io/nz-planner/dev/
- **Branch convention:** `dev` = in-progress, `main` = stable/shareable
- **File:** always `index.html` at repo root
- **Claude Code handoff:** "Push the latest planner as index.html to the dev branch" / "Merge dev into main and push"

---

## Supabase Credentials

- **Project URL:** `https://szcmxrqehkrbeolxwbng.supabase.co`
- **Publishable key:** `sb_publishable_Q9yXwZ2ax8OLhSDdEuoM8A_QkbUTDYs`
- **JWT anon key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InN6Y214cnFlaGtyYmVvbHh3Ym5nIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzI4NzU0OTQsImV4cCI6MjA4ODQ1MTQ5NH0.HV6-1VANpOwvE-GK9GlsfNIU2Fs4Ibph0kXxTSLWoCk`
- **Table:** `plans` (columns: `code TEXT PK`, `state JSONB`, `updated_at TIMESTAMPTZ`, `updated_by TEXT`)
- **RLS:** public read/write policy enabled
- **Auth headers:** `apikey` = publishable key; `Authorization: Bearer` = JWT anon key

---

## Technical Stack

- Single HTML file — all CSS and JS inline, zero build tooling
- Vanilla JavaScript, no frameworks
- **Leaflet.js 1.9.4** — route map (loaded dynamically from cdnjs)
- **OSRM** — live local drive time calculations (`router.project-osrm.org`)
- **Supabase** — real-time plan persistence
- **GitHub Pages** — static hosting
- **Google Fonts** — DM Serif Display, DM Mono, DM Sans

---

## State Serialisation Schema

```javascript
{
  tripConfig: {
    startDate: 'YYYY-MM-DD',          // default '2026-12-18'
    numDays: 44,
    arrivalAirport: 'AKL',            // sets the start zone (see AIRPORT_ZONES)
    arrivalNote: 'Arrive Auckland — pick up campervan',
    flightTime: '',                   // optional
    flightDest: '',                   // optional
    startAirport: 'CHC',              // departure airport for the flight home — sets the end zone
    camperNote: 'Return campervan in Christchurch',
    lockedDays: {},                   // no longer used — Kepler lock removed
    tripName: 'NZ North + South',
    niTargetDays: 18,                 // North Island day target shown in the summary
    selfContained: false,             // affects DOC freedom camp visibility
  },
  scheduled:        { 'day-N': ['activity-id', ...] },
  driveEve:         { 'day-N': true },
  selectedCamp:     { 'day-N': campIndex },
  expandedDays:     ['day-0', ...],
  customActivities: [...],
  userNotes:        { activityId: 'text' },
  booked:           { activityId: true },   // NEW — book-ahead checklist ticks (synced)
}
```

### localStorage keys

The v2 planner uses **separate keys** so it never overwrites the March South Island plan:

| Key | Purpose |
|---|---|
| `nz-planner-v2-code` | Saved sync passcode (was `nz-planner-code`) |
| `nz-planner-v2-configured` | Setup wizard completed flag (was `nz-planner-configured`) |

All access goes through `lsGet` / `lsSet` / `lsDel`, which swallow storage errors.

---

## Features (current dev build)

### Setup Wizard
- 2-step wizard; shown on first load (`nz-planner-v2-configured`) and via ⚙ setup button
- **Step 1:** start date, number of days (5–90), arrival airport, North Island target days, arrival note
- **Step 2:** departure airport, flight destination, departure time, camper note, self-contained checkbox
- Header title is derived from the start/end islands: "North → South", "South → North", "North Island" or "South Island"

### Start & End Zones
- `AIRPORT_ZONES` maps airport codes to planner zones:
  AKL→Auckland, WLG→Wellington, CHC→CHC, ZQN→Queenstown, NSN→Nelson, DUD→Dunedin, NPE→Napier, ROT→Rotorua, TRG→Tauranga, NPL→Taranaki, BHE→Blenheim, HKK→Hokitika, WSZ→Westport
- `startZone()` (default Auckland) is the zone before day 1; `endZone()` (default CHC) is the fly-day destination
- Day 1 shows an ✈ arrival item; the fly day shows `✈ Fly <dep> → <dest|home> (<time>)` plus drive to the end zone

### Three-Tab Layout
| Tab | Key |
|---|---|
| 📋 Plan | Day cards, fatigue banner, trip summary, effort chart, book-ahead checklist |
| 🚧 Roads | Static road condition cards, grouped by island (see Roads section) |
| 🗺 Map | Leaflet route map — initialised lazily on first switch |

### Day Cards
- Mobile-first vertical layout, expand/collapse on tap
- Load bar: green <70%, amber 70–90%, red >90% of 14 h max
- Public holidays labelled in gold next to the date
- Ferry days show `⛴ Xh transit incl. ferry`
- 📌 `book` tag on activities that need advance booking
- Multi-day tracks auto-select their overnight hut/camp via `campHint`
- **No locked days** — all days freely editable

### Activity Catalogue — 207 activities across 49 zones
- 110 North Island, 97 South Island (incl. the ferry item)
- See Activity Index below

### Campsite System
- 50 zones have curated campsite options: holiday-park, doc-standard, doc-freedom, hut
- DOC freedom camps hidden when `selfContained = false`
- Per-day campsite selector (multi-option toggle buttons); "book" link only shown when a site has a link
- Camp coordinates used as OSRM sleep-start waypoint
- Fox Glacier falls back to Franz Josef campsites

### Bottom Sheet
- **Type filter chips:** all, hike, climb, paddle, run, wildlife, rec, new, easy, **bike**, **book ahead**, **LOTR**
- **Island filter chips:** Both islands / North Island / South Island (`state.islandFilter`)
- Zone groups sorted by `ZONE_ORDER`, with "North Island" / "South Island" dividers
- Search by name or zone
- First tap = expand; second tap = place (or enter place mode)
- Already-scheduled activities shown at 35% opacity
- Expanded rows show `📌 Book: <bookNote>` when relevant

### Custom Activities
- Create / edit / delete via inline form in bottom sheet
- Zone dropdown is built from `ZONE_ORDER` (only zones with coordinates)
- Types: hike, climb, paddle, run, **bike**, wildlife, scenic, soak
- Hard flag auto-computed: gain >800 m OR actHrs >8
- Persist in Supabase state

### Day Load Calculation
| Component | Source |
|---|---|
| Transit drive | `getDrive()` — DRIVE table, then shortest path through the table, then haversine fallback |
| Local drive | OSRM live (sleep start → activity coords → sleep end); fallback: max driveFromBase × 2 |
| Evening drive | Transit for next day's zone if driveEve toggled |
| Activity hours | Sum of actHrs |
| **Hard day** | gain >800 m OR total hours **>10** |

### Drive Time Routing
- Direct lookup in `DRIVE` (bidirectional)
- If no direct entry: Dijkstra over the DRIVE graph (results cached in `_driveCache`), so e.g. Auckland → Napier or Taupō → Picton (via the ferry) still return sensible times
- Zones in `NO_THROUGH_ZONES` (currently `Heaphy` — walk-in only) are never used as an intermediate road waypoint
- Last resort: haversine × 1.4 at 70 km/h, rounded to 0.25 h

### Cook Strait Ferry
- `ni-ferry` activity (zone Picton): Wellington → Picton, `actHrs: 0`; sailing + check-in is counted as transit (DRIVE Wellington↔Picton = 4.5 h)
- `dayCrossesStrait()` detects any day whose transit path changes island
- Crossing day **with** the ferry item: info — keep the next day flexible for southerly cancellations
- Crossing day **without** it: warning to add the ferry and book vehicle space; also flagged in the book-ahead checklist
- Ferry item doesn't count as a second zone for the multi-zone warning

### Public Holidays
- `getHoliday(date)` computes NZ holidays for any year, with Mondayisation:
  - Christmas Day, Boxing Day, New Year's Day, 2 Jan (+ observed days)
  - Regional: Wellington Anniversary (Mon nearest 22 Jan), Auckland Anniversary (Mon nearest 29 Jan, also Northland), Nelson Anniversary (Mon nearest 1 Feb)
- Shown as info on every day, including empty ones; Christmas Day gets a "restricted trading — stock up on the 24th" note
- On Christmas Day, each `book` activity gets a "check it operates" prompt

### Fatigue & Warnings (per day)
- Public holiday: info
- 3+ consecutive hard days: warn
- Day total >13 h: "very overloaded" warn; >10 h: "long day" warn
- Road transit >4 h: warn; >2.5 h: info (ferry time excluded, noted as "plus the ferry")
- Multi-zone in single day: warn (ferry excluded)
- Operator not running that weekday (`noDays` / `noDaysMsg`, e.g. Tiritiri Matangi Mon/Tue) — suppressed on public holidays: warn
- Backtrack: thisZone >2 steps earlier in ZONE_ORDER than prevZone (flex zones exempt)
- Stewart Island: ferry / leave-the-camper-at-Bluff logistics info
- Cook Strait crossing: see above

### Book-Ahead Checklist
- Collapsible card above the day cards, shown when any scheduled activity has `book: true` (or a crossing day lacks the ferry)
- One row per bookable activity in date order: date, name, `bookNote`, link, checkbox
- Ticks stored in `state.booked` and synced via Supabase
- Opened from the "To book ↗" summary tile

### Trip Summary
| Tile | Metric |
|---|---|
| Total Drive | Hours + 🧙 Gandalf days (drive × 2 / 10) |
| Elevation | Partial emoji progress bar — total gain / 2291 m (Mt Ngauruhoe) |
| Quest Distance | Total km + LOTR milestone (% of 1,800 km Shire→Mount Doom) |
| Hard Days | Count of hard days |
| **Island days** | `N · S` day counts, ⛴ crossing days, and progress against `niTargetDays` |
| **To book** | Outstanding bookings · `x of y booked` (tap to open checklist) |

Island days: each day is N, S or X (crossing). Empty days take the island you're on; the fly day takes the end zone's island.

### LOTR Quest Modal
- 11 milestones: The Shire → Mount Doom
- Unlocked by total trip distance as % of 1,800 km
- Filming-location activities are tagged `lotr: true` (Hobbiton, Tongariro Crossing, Mangawhero Falls, Wētā Workshop, Mt Victoria, Kaitoke/Rivendell, Putangirua Pinnacles, Ōpārara/Moria Gate)

### Effort Bar Chart
- Mini bar chart, one bar per day, scaled to max effort score
- Colours: green ≤2, amber ≤5, orange ≤7, red >7

### Real-Time Sync
- 4-letter passcode (24-char alphabet, no I/O)
- Poll: every 3 s; save: debounced 800 ms
- Conflict: last-write-wins; self-writes ignored via deviceId check
- Auto-reconnect from localStorage on load; saves blocked until reconnect completes
- Loading remote state also refreshes the header title (start/end island)

### Route Map
- CartoDB dark tiles, Leaflet 1.9.4; default view centred on the whole country (−41.0, 173.5, zoom 5)
- Route line: start airport → numbered zone pins (first scheduled occurrence) → end airport, dashed green
- Square markers: blue = start, gold = end
- Zone coords from `ZONE_COORDS`, falling back to `BASE_COORDS` (`zc()`)
- Graceful offline degradation

### Roads Tab
**North Island — your route**
- ⛴ Cook Strait ferry (Interislander / Bluebridge) — book vehicle space 4–8+ weeks out
- SH35 Pacific Coast Highway / East Cape — 2026 storm damage, restricted hours
- SH38 Te Urewera — ~27 km unsealed, check rental terms
- SH25 Coromandel — slips, gravel north of Colville
- SH1/47/48 Desert Road & Tongariro — no Crossing parking at Mangatepopo
- SH3 Awakino Gorge / Mt Messenger — bypass works; SH43 alternative
- SH1 Northland — Brynderwyns, SH12, no Ninety Mile Beach in rentals
- 🎄 Christmas / New Year holiday traffic

**South Island — north-west & West Coast**
- SH60 Tākaka Hill, SH6/67 Buller Gorge & Karamea, SH7 Lewis Pass

**South Island — other roads (from the March trip)**
- The original March-trip cards (Homer Tunnel / SH94, Haast Pass, Arthur's Pass, the long Te Anau → Haast day, etc.); text updated for summer (Dec–Jan)

Live-status link row now covers Northland, Auckland, Waikato, Bay of Plenty, Hawke's Bay, Taranaki, Manawatū-Whanganui, Gisborne/Wellington plus the South Island regions.

---

## Zone Route Order (54 zones)

```
North Island:
Auckland → Whangarei → Bay of Islands → Far North → Kauri Coast → Thames → Coromandel Town →
Hahei → Tauranga → Rotorua → East Cape → Gisborne → Waikaremoana → Napier → Taupo →
Tongariro → Wharepapa → Raglan → Waitomo → Taranaki → Wairarapa → Wellington

South Island:
Picton → Marlborough Sounds → Blenheim → Nelson → Marahau → Takaka → Heaphy → Nelson Lakes →
Karamea → Westport → Punakaiki → Hokitika → Franz Josef → Fox Glacier → Haast → Wanaka →
Glenorchy → Queenstown → Te Anau → Milford → Bluff → Stewart Island → Dunedin → Mt Cook →
Mackenzie → Peel Forest → Lewis Pass → Hanmer → Kaikoura → Arthurs Pass → Castle Hill → CHC
```

- **North Island zones** are listed in `NI_ZONES`; `islandOf(zone)` returns `'N'` or `'S'`
- **Flex zones** (no backtrack warning): CHC, Castle Hill, Arthurs Pass, Hanmer, Lewis Pass, Auckland, Wellington, Picton
- Zones with no activities (transit / campsite only): Blenheim, Nelson, Milford, Bluff, CHC

---

## Activity Index (207 activities)

Zone strings use ASCII — no macrons (`'Kaikoura'` not `'Kaikōura'`, `'Taupo'` not `'Taupō'`).
ID prefixes: `ni-` = North Island (plus the ferry), `si-` = South Island NW additions (Oct 2026), unprefixed = March 2026 South Island catalogue.
Full details (coords, dist, gain, hours, notes, links) live in `ACTIVITIES`, `NI_ACTIVITIES` and `SI_NW_ACTIVITIES` in index.html.

Markers: ¹ hard · ² book ahead · ³ LOTR

### North Island (110)
| Zone | # | Activity IDs |
|---|---|---|
| Auckland | 9 | `ni-rangitoto`², `ni-tiritiri`², `ni-piha`, `ni-karekare`, `ni-muriwai`, `ni-wainamu`, `ni-goat-island`, `ni-waiheke`, `ni-auckland-rest` |
| Whangarei | 5 | `ni-mangawhai`, `ni-waipu-caves`, `ni-bream-head`, `ni-poor-knights`², `ni-abbey-caves` |
| Bay of Islands | 6 | `ni-cape-brett-1`¹², `ni-cape-brett-2`¹, `ni-urupukapuka`², `ni-waitangi`, `ni-boi-cruise`², `ni-kerikeri` |
| Far North | 2 | `ni-cape-reinga`, `ni-te-paki-dunes` |
| Kauri Coast | 4 | `ni-tane-mahuta`, `ni-trounson-kiwi`, `ni-kai-iwi`, `ni-hokianga` |
| Thames | 2 | `ni-pinnacles`¹², `ni-karangahake` |
| Coromandel Town | 3 | `ni-coastal-walkway`, `ni-new-chums`, `ni-driving-creek`² |
| Hahei | 4 | `ni-cathedral-cove`, `ni-cathedral-kayak`², `ni-hot-water-beach`, `ni-tairua` |
| Tauranga | 2 | `ni-mauao`, `ni-mclaren-glowworm`² |
| Rotorua | 12 | `ni-redwoods-mtb`, `ni-redwoods-run`, `ni-waiotapu`, `ni-tarawera-trail`², `ni-kaituna`², `ni-mt-tarawera`², `ni-waimangu`, `ni-whakarewarewa`, `ni-rotorua-rest`, `ni-blue-lake`, `ni-whirinaki`, `ni-hobbiton`²³ |
| East Cape | 2 | `ni-east-cape-lh`, `ni-pacific-coast` |
| Gisborne | 2 | `ni-rere`, `ni-cooks-cove` |
| Waikaremoana | 5 | `ni-waikaremoana-1`², `ni-waikaremoana-2`, `ni-waikaremoana-3`, `ni-panekire-day`, `ni-waikareiti` |
| Napier | 5 | `ni-gannet-safari`², `ni-kidnappers-walk`, `ni-te-mata`, `ni-hb-trails`, `ni-napier-rest` |
| Taupo | 5 | `ni-huka-trail`, `ni-mine-bay`², `ni-kawakawa-bay`¹, `ni-great-lake-trail`, `ni-taupo-rest` |
| Tongariro | 12 | `ni-tac`¹²³, `ni-tama-lakes`, `ni-taranaki-falls`, `ni-silica-rapids`, `ni-ruapehu-crater`¹, `ni-waitonga-mangawhero`³, `ni-old-coach-road`, `ni-whanganui-1`², `ni-whanganui-2`, `ni-whanganui-3`, `ni-bridge-to-nowhere`², `ni-tnc`¹² |
| Wharepapa | 3 | `ni-froggatt`¹, `ni-waipari`, `ni-maungatautari` |
| Raglan | 3 | `ni-raglan-surf`², `ni-karioi`, `ni-bridal-veil` |
| Waitomo | 3 | `ni-black-water`², `ni-ruakuri-night`, `ni-marokopa` |
| Taranaki | 5 | `ni-pouakai-crossing`¹², `ni-pouakai-tarns`², `ni-taranaki-summit`¹, `ni-dawson-wilkies`, `ni-np-coastal` |
| Wairarapa | 6 | `ni-putangirua`³, `ni-cape-palliser`, `ni-holdsworth`¹, `ni-martinborough`, `ni-pukaha`, `ni-remutaka` |
| Wellington | 10 | `ni-kapiti-1`², `ni-kapiti-2`, `ni-kapiti-day`², `ni-zealandia`², `ni-weta`²³, `ni-mt-vic`³, `ni-wellington-rest`, `ni-red-rocks`, `ni-matiu`, `ni-kaitoke`³ |

### South Island (97)
| Zone | # | Activity IDs |
|---|---|---|
| Picton | 1 | `ni-ferry`² (Cook Strait crossing) |
| Marlborough Sounds | 4 | `queen-charlotte-day`, `tirohanga-picton`, `qc-sound-kayak`, `qct-ridge-torea-anakiwa` |
| Marahau | 2 | `at-kayak`, `at-walk` |
| Takaka | 11 | `paynes-ford`¹, `golden-bay`, `wharariki-beach`, `cape-farewell-horses`, `farewell-spit`, `abel-tasman-north-kayak`, `tablelands-kahurangi`¹, `mt-arthur-summit`¹, `si-pupu-springs`, `si-rawhiti-cave`, `si-totaranui` |
| Heaphy | 4 | `si-heaphy-1`², `si-heaphy-2`, `si-heaphy-3`, `si-heaphy-4` |
| Nelson Lakes | 6 | `mt-robert-circuit`, `st-arnaud-range`¹, `lake-rotoiti-circuit`, `robert-ridge-angelus`¹, `travers-valley`, `pelorus-bridge` |
| Karamea | 2 | `si-scotts-beach`, `si-oparara`³ |
| Westport | 4 | `si-charming-creek`, `si-cape-foulwind`, `si-denniston`, `si-ogr-lyell`¹ |
| Punakaiki | 3 | `paparoa`, `pororari-gorge`, `si-paparoa-gw`¹² |
| Hokitika | 4 | `mt-brown-1`¹, `mt-brown-2`, `hokitika-gorge`, `si-wilderness-trail` |
| Franz Josef | 5 | `glaciers`, `fox-peak`¹, `alex-knob`¹, `okarito-kayak`, `okarito-kiwi` |
| Fox Glacier | 1 | `fox-chancellor`¹ |
| Haast | 3 | `haast-pass`, `wellcome-flat-1`, `wellcome-flat-2` |
| Wanaka | 7 | `hospital-flat`¹ (×2 — see Known Issues), `roys-peak`¹, `rob-roy`, `brewster`¹, `rest-wanaka`, `isthmus-peak`¹ |
| Glenorchy | 4 | `routeburn-run`¹, `earnslaw`¹, `ben-lomond`¹, `glenorchy-lagoon` |
| Queenstown | 1 | `rest-queenstown` |
| Te Anau | 7 | `key-summit`, `gertrude`¹, `milford-kayak`, `kepler`¹, `kepler2`¹, `dusky-track`¹, `glowworms` |
| Stewart Island | 3 | `ulva`, `rakiura`, `kiwi` |
| Dunedin | 2 | `sandfly-bay-penguins`, `moeraki-boulders` |
| Mt Cook | 6 | `hooker`, `mueller`¹, `sebastopol`, `mt-wakefield`¹, `upper-hopkins`, `upper-hopkins-2` |
| Mackenzie | 3 | `tekapo-stargazing`, `tekapo-lake-alexandrina`, `lake-pukaki` |
| Peel Forest | 1 | `peel-forest` |
| Lewis Pass | 3 | `si-lake-daniell`, `si-maruia-springs`, `si-st-james`¹ |
| Hanmer | 1 | `hanmer` |
| Kaikoura | 6 | `whale-watch`, `dolphin`, `kaikoura-peninsula`, `mt-fyffe`¹, `ohau-seals`, `mt-fyffe-overnight`¹ |
| Arthurs Pass | 2 | `avalanche-peak`¹, `temple-kelly`¹ |
| Castle Hill | 1 | `castle-hill` |

### Multi-day groups
`upper-hopkins`, `wellcome-flat`, `mt-brown`, `cape-brett`, `whanganui` (3-day Whanganui Journey canoe), `waikaremoana` (3-day Great Walk), `kapiti` (overnight), `heaphy` (4-day Great Walk)

### Notable changes to existing South Island activities
- Notes rewritten for a Dec–Jan summer trip (snow, water temps, closures)
- `kepler` / `kepler2`: no longer pre-booked — the March 2026 booking doesn't carry over; needs a new Great Walk booking
- "Jucy" references removed — vehicle is now a generic campervan

---

## Recipe for Adding Activities

```javascript
{ id: 'ni-unique-id', coords: [-lat, lng], name: 'Display Name',
  zone: 'Zone Name', baseZone: 'Zone Name',   // ASCII, must exist in BASE_COORDS / DRIVE
  types: ['hike'],                             // hike, climb, paddle, run, bike, wildlife, scenic, soak
  dist: 10, gain: 400, actHrs: 4,
  driveFromBase: 0.5,                          // omit if 0
  link: 'https://...',
  notes: 'Notes here.',
  hard: true, rec: false, isNew: true,
  // optional:
  book: true, bookNote: 'Book 4+ weeks out at …',  // shows in the Book-ahead checklist
  lotr: true,                                      // LOTR filming location
  noDays: [1, 2], noDaysMsg: 'Doesn’t sail Mon/Tue', // JS getDay() values with no service
  campHint: 'Panekire Hut',                        // substring of the campsite name to pre-select
}
```

- North Island activities go in `NI_ACTIVITIES`; new South Island ones in `SI_NW_ACTIVITIES` (both are pushed into `ACTIVITIES`)
- Multi-day: add `multiDay:true, multiDayPart:1, multiDayTotal:2, multiDayGroup:'group-id'` (and a `campHint` for each night's hut)
- New zone: add it to `BASE_COORDS`, `CAMPSITES`, at least one `DRIVE` edge (in `addDriveTimes`), `ZONE_ORDER`, and `NI_ZONES` if it's on the North Island
- Zone strings: ASCII only, no macrons
- `hard` flag is manual — set if gain >800 m or the total day is likely to breach 10 h

---

## Drive Time Table (key pairs)

Original South Island pairs are in the `DRIVE` object; new pairs are added in `addDriveTimes()`. All times bidirectional, in hours. Pairs not listed are routed through the table (see Drive Time Routing).

### North Island
| From | To | Hours |
|---|---|---|
| Auckland | Whangarei | 2.00 |
| Auckland | Bay of Islands | 3.25 |
| Auckland | Thames | 1.25 |
| Auckland | Rotorua | 3.00 |
| Auckland | Waitomo | 2.50 |
| Auckland | Taupo | 3.50 |
| Whangarei | Bay of Islands | 1.00 |
| Bay of Islands | Far North | 2.60 |
| Bay of Islands | Kauri Coast | 1.75 |
| Thames | Coromandel Town | 1.00 |
| Thames | Hahei | 1.25 |
| Hahei | Tauranga | 2.25 |
| Tauranga | Rotorua | 1.00 |
| Rotorua | Taupo | 1.00 |
| Rotorua | East Cape | 4.75 |
| Rotorua | Waikaremoana | 3.50 |
| East Cape | Gisborne | 2.75 |
| Gisborne | Napier | 3.00 |
| Waikaremoana | Napier | 2.75 |
| Napier | Taupo | 1.75 |
| Taupo | Tongariro | 1.25 |
| Tongariro | Waitomo | 2.00 |
| Tongariro | Wellington | 4.25 |
| Wharepapa | Waitomo | 1.00 |
| Raglan | Waitomo | 1.25 |
| Waitomo | Taranaki | 2.50 |
| Taranaki | Wellington | 4.50 |
| Wairarapa | Wellington | 1.25 |
| **Wellington** | **Picton (ferry)** | **4.50** (sailing + check-in) |

### South Island
| From | To | Hours |
|---|---|---|
| CHC | Castle Hill | 1.25 |
| CHC | Arthurs Pass | 1.75 |
| CHC | Hanmer | 1.50 |
| Castle Hill | Mt Cook | 2.75 |
| Castle Hill | Arthurs Pass | 0.50 |
| Mt Cook | Wanaka | 2.25 |
| Wanaka | Glenorchy | 1.50 |
| Wanaka | Franz Josef | 3.83 |
| Glenorchy | Te Anau | 2.75 |
| Te Anau | Milford | 2.50 |
| Te Anau | Bluff | 2.60 |
| Bluff | Stewart Island (ferry) | 0.25 |
| Bluff | Haast | 6.50 |
| Haast | Franz Josef | 1.50 |
| Franz Josef | Fox Glacier | 0.40 |
| Franz Josef | Punakaiki | 3.00 |
| Punakaiki | Marahau | 2.50 |
| Marahau | Kaikoura | 4.50 |
| Kaikoura | Hanmer | 2.00 |
| Hanmer | CHC | 1.50 |
| Nelson Lakes | CHC | 5.00 |
| Nelson Lakes | Marahau | 2.50 |
| Nelson Lakes | Picton | 2.00 |
| Blenheim | CHC | 1.50 |
| Picton | Blenheim | 0.50 |
| Mackenzie | Mt Cook | 1.25 |
| Peel Forest | CHC | 2.00 |
| Takaka | Heaphy | 1.00 |
| Heaphy | Karamea | 0.25 (walk-in zone — not a through-route) |
| Nelson | Picton | 2.00 |
| Nelson | Westport | 3.25 |
| Westport | Karamea | 1.50 |
| Westport | Punakaiki | 0.75 |
| Punakaiki | Greymouth | 0.75 |
| Greymouth | Hokitika | 0.50 |
| Lewis Pass | Hanmer | 1.00 |
| Lewis Pass | CHC | 2.75 |

---

## Known Issues

- `hospital-flat` is defined twice in the Wanaka block (different coords) — `getAct()` returns the first, and the sheet lists it twice
- `lockedDays` is still written to `tripConfig` by the wizard but no longer used

---

## Conor's Working Style

- Provides corrections as numbered lists; expects all points actioned in a single pass
- Prefers detailed, data-rich outputs (tables, interactive artifacts, probability breakdowns)
- Field-level specificity over general advice
- When editing the planner HTML, always work from `/home/claude/planner_v3.html` (copy from `/mnt/project/index.html`) and copy output to `/mnt/user-data/outputs/index.html` when done
