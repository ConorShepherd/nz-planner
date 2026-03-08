# NZ South Island Trip Planner — Project Knowledge

> **Source of truth:** This document is reverse-engineered from `index.html` (production, March 2026).  
> Replace the old knowledge doc with this file.

---

## People & Trip Context

- **Travellers:** 2 people (Conor + partner)
- **Vehicle:** Jucy campervan — configurable as self-contained or not (affects campsite options)
- **Default trip:** 16 March – 4 April 2026 (20 days incl. fly day)
- **Default arrival:** CHC 08:30 Mon 16 Mar
- **Default departure:** CHC → MEL 15:45 Sat 4 Apr
- **Comfortable day:** 10–14 km / 400–600 m / 3–5 h
- **Big day:** 16–18 km / 800–1,400 m / 6–9 h (use ×0.80 on DOC times)
- **Hard day:** cumulative gain >800 m OR total committed hours >10 h
- **Max consecutive hard days:** 2–3 (flagged at 3+)

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
    startDate: 'YYYY-MM-DD',
    numDays: 20,
    flightTime: '15:45',
    flightDest: 'MEL',
    startAirport: 'CHC',
    camperNote: 'Return Jucy camper in morning',
    lockedDays: {},          // no longer used — Kepler lock removed
    tripName: 'NZ South Island',
    selfContained: false,    // affects DOC freedom camp visibility
  },
  scheduled:        { 'day-N': ['activity-id', ...] },
  driveEve:         { 'day-N': true },
  selectedCamp:     { 'day-N': campIndex },
  expandedDays:     ['day-0', ...],
  customActivities: [...],
  userNotes:        { activityId: 'text' },
}
```

---

## Features (current production build)

### Setup Wizard
- 2-step wizard; shown on first load (localStorage flag `nz-planner-configured`) and via ⚙ setup button
- Step 1: start date, number of days
- Step 2: departure airport, flight destination, departure time, camper note, self-contained checkbox

### Three-Tab Layout
| Tab | Key |
|---|---|
| 📋 Plan | Day cards, fatigue banner, trip summary, effort chart |
| 🚧 Roads | Static road condition cards (SH6, SH80, etc.) |
| 🗺 Map | Leaflet route map — initialised lazily on first switch |

### Day Cards
- Mobile-first vertical layout, expand/collapse on tap
- Load bar: green <70%, amber 70–90%, red >90% of 14 h max
- Fly day auto-calculates drive-to-departure-airport from last scheduled zone
- **No locked days** — Kepler locking removed; all days freely editable

### Activity Catalogue — 53 activities across 13+ zones
(See Activity Data section below for full list)

### Campsite System
- Each zone has curated campsite options: holiday-park, doc-standard, doc-freedom, hut
- DOC freedom camps hidden when `selfContained = false`
- Per-day campsite selector (multi-option toggle buttons)
- Camp coordinates used as OSRM sleep-start waypoint

### Bottom Sheet
- Filter chips: all, hike, climb, paddle, run, wildlife, rec, new, easy
- Search by name or zone
- First tap = expand; second tap = place (or enter place mode)
- Already-scheduled activities shown at 35% opacity

### Custom Activities
- Create / edit / delete via inline form in bottom sheet
- Hard flag auto-computed: gain >800 m OR actHrs >8
- Persist in Supabase state

### Day Load Calculation
| Component | Source |
|---|---|
| Transit drive | DRIVE lookup table (bidirectional) |
| Local drive | OSRM live (sleep start → activity coords → sleep end); fallback: max driveFromBase × 2 |
| Evening drive | Transit for next day's zone if driveEve toggled |
| Activity hours | Sum of actHrs |
| **Hard day** | gain >800 m OR total hours **>10** |

### Fatigue & Warnings (per day)
- 3+ consecutive hard days: warn
- Day total >13 h: "very overloaded" warn; >10 h: "long day" warn
- Transit >4 h: warn; >2.5 h: info
- Multi-zone in single day: warn
- Backtrack: thisZone >2 steps earlier in ZONE_ORDER than prevZone (flex zones exempt)
- Stewart Island: always shows ferry/camper logistics info

### Trip Summary (2×2 grid)
| Tile | Metric |
|---|---|
| Total Drive | Hours + 🧙 Gandalf days (drive × 2 / 10) |
| Elevation | Partial emoji progress bar — total gain / 2291 m (Mt Ngauruhoe) |
| Quest Distance | Total km + LOTR milestone (% of 1,800 km Shire→Mount Doom) |
| Hard Days | Count of hard days |

### LOTR Quest Modal
- 11 milestones: The Shire → Mount Doom
- Unlocked by total trip distance as % of 1,800 km

### Effort Bar Chart
- Mini bar chart, one bar per day, scaled to max effort score
- Colours: green ≤2, amber ≤5, orange ≤7, red >7

### Real-Time Sync
- 4-letter passcode (24-char alphabet, no I/O)
- Poll: every 3 s; save: debounced 800 ms
- Conflict: last-write-wins; self-writes ignored via deviceId check
- Auto-reconnect from localStorage on load; saves blocked until reconnect completes

### Route Map
- CartoDB dark tiles, Leaflet 1.9.4
- Numbered pins per zone (first scheduled occurrence), dashed green polyline
- Graceful offline degradation

---

## Zone Route Order (25 zones)

```
CHC → Castle Hill → Arthurs Pass → Peel Forest → Mackenzie → Mt Cook → Wanaka →
Glenorchy → Queenstown → Te Anau → Milford → Bluff → Stewart Island → Haast →
Franz Josef → Fox Glacier → Hokitika → Punakaiki → Marahau → Takaka → Nelson Lakes →
Picton → Blenheim → Kaikoura → Hanmer
```

**Flex zones** (no backtrack warning): CHC, Castle Hill, Arthurs Pass, Hanmer

---

## Activity Data (53 activities)

Zone strings use ASCII — no macrons (`'Kaikoura'` not `'Kaikōura'`, `'Wanaka'` not `'Wānaka'`).

### Castle Hill / Mt Cook
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| castle-hill | Castle Hill Bouldering | climb | 0 | 0 | 4 | — | Arrival day. No gear hire. |
| hooker | Hooker Valley Track | hike | 10 | 145 | 2.5 | — | Sunrise. Bridge may be closed. |
| mueller | Mueller Hut Route | hike | 10.5 | 1045 | 5.5 | ✓ REC | Register at visitor centre. |
| sebastopol | Sebastopol Bluffs Cragging | climb | 0 | 0 | 3 | — REC NEW | Roadside schist. |
| mt-wakefield | Mt Wakefield Day Walk / Wild Camp | hike | 9 | 750 | 4 | ✓ REC NEW | Unmarked above bushline. |
| upper-hopkins | Upper Hopkins Day 1 | hike | 24 | 400 | 6 | — REC NEW ↕1/2 | +1.0h drive. Memorial Hut. |
| upper-hopkins-2 | Upper Hopkins Day 2: Return | hike | 24 | 400 | 5 | — REC NEW ↕2/2 | +1.0h drive. |

### Wanaka
| ID | Name | Types | Dist | Gain | Hrs | Hard | Local drive |
|---|---|---|---|---|---|---|---|
| hospital-flat | Hospital Flat — Sport Climbing | climb | 0 | 0 | 5 | ✓ | +0.5h |
| roys-peak | Roy's Peak | hike | 16 | 1228 | 4.5 | ✓ | +0.25h |
| rob-roy | Rob Roy Glacier Track | hike | 10 | 450 | 2.75 | — NEW | +1.0h |
| brewster | Brewster Hut Track | hike | 10 | 1200 | 4.5 | ✓ NEW | +1.25h. En route Haast. |

### Glenorchy / Queenstown
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| routeburn-run | Routeburn — Harris Saddle Day Run | run | 28 | 1100 | 7 | ✓ | +0.75h |
| earnslaw | Mt Earnslaw / Sentinel Peak | hike | 8 | 900 | 4.5 | ✓ NEW | +0.33h |
| ben-lomond | Ben Lomond, Queenstown | hike | 11 | 1438 | 4.5 | ✓ | baseZone: Queenstown. Gondola. |

### Te Anau / Fiordland
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| key-summit | Key Summit Day Walk | hike | 8 | 400 | 3.5 | — REC | +1.5h drive. baseZone: Te Anau. |
| gertrude | Gertrude Saddle | hike | 10 | 700 | 4.5 | ✓ REC NEW | +1.33h. Best Fiordland day hike. |
| milford-kayak | Milford Sound — Kayaking | paddle | 0 | 0 | 5 | — | +2.5h. Rosco's. |
| kepler | KEPLER Day 1: Control Gates → Iris Burn | hike | 27 | 800 | 8 | ✓ | No longer locked. PRE-BOOKED Mon 23 Mar. |
| kepler2 | Kepler Day 2: Iris Burn → Control Gates | hike | 30 | 600 | 7 | ✓ | Should follow Day 1. |
| dusky-track | Dusky Track (multi-day reference) | hike | 84 | 5000 | 40 | ✓ REC NEW | 8–10 days. Listed as inspiration only. |
| glowworms | Te Anau Glowworm Caves | scenic | 0 | 0 | 2.5 | — | Real Journeys. Book ahead. |

### Stewart Island
| ID | Name | Types | Dist | Gain | Hrs | Notes |
|---|---|---|---|---|---|---|
| ulva | Ulva Island — Open Sanctuary | wildlife/hike | 5 | 50 | 3 | +0.17h water taxi. |
| rakiura | Rakiura Track Section | hike | 12 | 200 | 3 | Gaiters recommended. |
| kiwi | Kiwi Spotting — Dusk Walk | wildlife | 0 | 0 | 1.5 | Near-guaranteed. Self-guided or Ruggedy Range. NEW |

### West Coast / Haast
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| haast-pass | Haast Pass Roadside Stops | scenic/hike | 3 | 50 | 1.5 | — NEW | Thunder Creek, Blue Pools. |
| wellcome-flat-1 | Wellcome Flat Day 1 | hike | 17 | 200 | 4 | — REC NEW ↕1/2 | +0.5h. Book huts.co.nz. |
| wellcome-flat-2 | Wellcome Flat Day 2 | hike/soak | 17 | 200 | 4 | — REC NEW ↕2/2 | +0.5h. Morning soak. |
| glaciers | Glacier Valley Walks (Fox & Franz Josef) | scenic | 4 | 100 | 2 | — | Valley floor only — no glacier access. |
| fox-peak | Fox Peak Day Walk | hike | 12 | 1100 | 5 | ✓ REC NEW | +1.5h drive. 4WD trailhead — check clearance. |
| alex-knob | Alex Knob Track (Franz Josef) | hike | 17 | 1200 | 5 | ✓ NEW | Clear weather only. |

### Hokitika
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| mt-brown-1 | Mt Brown Day 1 | hike | 7 | 900 | 3 | ✓ REC NEW ↕1/2 | +0.5h. Friend highlight. |
| mt-brown-2 | Mt Brown Day 2: Styx River Loop | hike | 7 | 0 | 2.5 | — REC NEW ↕2/2 | +0.5h. |

### Punakaiki
| ID | Name | Types | Dist | Gain | Hrs | Notes |
|---|---|---|---|---|---|---|
| paparoa | Paparoa Track Section + Pancake Rocks | hike/climb | 10 | 400 | 3.5 | REC NEW |

### Marahau / Abel Tasman / Takaka
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| at-kayak | Abel Tasman — Sea Kayaking | paddle | 0 | 0 | 5 | — | Water taxi to Awaroa recommended. |
| at-walk | Abel Tasman Coastal Track Run/Walk | run/hike | 26 | 400 | 5 | — | |
| paynes-ford | Payne's Ford — Climbing + DWS | climb | 0 | 0 | 6 | ✓ REC NEW | +0.5h. World-class marble. |
| golden-bay | Golden Bay Beaches + Tākaka Town | scenic | 0 | 0 | 4 | — REC NEW | Best rest day in north. |

### Kaikōura
| ID | Name | Types | Hrs | Notes |
|---|---|---|---|---|
| whale-watch | Whale Watch Kaikōura | wildlife | 3.5 | BOOK WEEKS AHEAD. |
| dolphin | Dolphin Encounter Kaikōura | wildlife | 3 | Swim with dusky dolphins. |
| kaikoura-peninsula | Kaikōura Peninsula Walkway | hike | 2.25 | 12 km, 150 m. Fur seals. NEW |
| mt-fyffe | Mt Fyffe Summit | hike | 6 | 16 km, 1602 m. HARD. +0.17h. NEW |

### Arthur's Pass / Hanmer
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| avalanche-peak | Avalanche Peak | hike | 10 | 1100 | 5 | ✓ REC | Best in settled weather only. |
| temple-kelly | Temple / Kelly Range / Carroll Hut | hike | 12 | 1000 | 5 | ✓ REC NEW | |
| hanmer | Hanmer Springs Thermal Pools | soak | 0 | 0 | 2 | — | End-of-trip ritual. |

### Nelson Lakes
| ID | Name | Types | Dist | Gain | Hrs | Hard | Notes |
|---|---|---|---|---|---|---|---|
| mt-robert-circuit | Mount Robert Circuit | hike | 9 | 635 | 3.5 | — NEW | +0.17h. Kea Hut. |
| st-arnaud-range | St Arnaud Range Track | hike | 11 | 1068 | 5 | ✓ NEW | Poled alpine route. |
| lake-rotoiti-circuit | Lake Rotoiti Circuit | hike | 23 | 300 | 6 | — NEW | Water taxi option. |

### Marlborough Sounds
| ID | Name | Types | Dist | Gain | Hrs | Notes |
|---|---|---|---|---|---|---|
| queen-charlotte-day | Queen Charlotte — Ship Cove to Furneaux Lodge | hike | 17 | 527 | 4 | +0.25h. Water taxi. NEW |
| tirohanga-picton | Tirohanga Track, Picton | hike | 7 | 350 | 2.5 | NEW |

### Mackenzie / Peel Forest
| ID | Name | Types | Dist | Gain | Hrs | Notes |
|---|---|---|---|---|---|---|
| peel-forest | Peel Forest Walks + DOC Camp | hike/scenic | 8 | 200 | 3 | NEW. Great stopover. |
| tekapo-stargazing | Lake Tekapo Stargazing | scenic | 0 | 0 | 2 | Dark Sky Reserve. Book Earth & Sky. NEW |
| tekapo-lake-alexandrina | Lake Alexandrina + Mt John Walk | hike/scenic | 10 | 350 | 3.5 | +0.17h. Hidden gem camp. NEW |
| lake-pukaki | Lake Pukaki — Glacier Viewpoint | scenic | 2 | 50 | 1 | On SH8. Essential stop. NEW |

### Rest Days
| ID | Name | Zone | Notes |
|---|---|---|---|
| rest-queenstown | Queenstown Rest Day | Queenstown | actHrs: 0. Full rest. |
| rest-wanaka | Wānaka Rest Day | Wanaka | actHrs: 0. Full rest. |

---

## Recipe for Adding Activities

```javascript
{ id: 'unique-id', coords: [-lat, lng], name: 'Display Name',
  zone: 'Zone Name', baseZone: 'Zone Name',   // ASCII, must match DRIVE table key
  types: ['hike'],                             // array
  dist: 10, gain: 400, actHrs: 4,
  driveFromBase: 0.5,                          // omit if 0
  link: 'https://...',
  notes: 'Notes here.',
  hard: true, rec: false, isNew: true }
```

- Multi-day: add `multiDay:true, multiDayPart:1, multiDayTotal:2, multiDayGroup:'group-id'`
- Zone strings: ASCII only, no macrons
- `hard` flag is manual — set if gain >800 m or the total day is likely to breach 10 h

---

## Drive Time Table (key pairs)

Full table is in the DRIVE object in index.html. Zone strings use ASCII. All times bidirectional.

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

---

## Conor's Working Style

- Provides corrections as numbered lists; expects all points actioned in a single pass
- Prefers detailed, data-rich outputs (tables, interactive artifacts, probability breakdowns)
- Field-level specificity over general advice
- When editing the planner HTML, always work from `/home/claude/planner_v3.html` (copy from `/mnt/project/index.html`) and copy output to `/mnt/user-data/outputs/index.html` when done
