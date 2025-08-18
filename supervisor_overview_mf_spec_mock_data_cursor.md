### Supervisor Overview — Medium‑Fidelity Mock (Spec)

This spec describes a medium‑fidelity **Supervisor Overview** UI for PhoneX Warehouse. It targets a dev sandbox (Cursor / Vite) with **React + TypeScript** and aims to look as polished as the earlier design screenshots. It includes visual rules, interactions, component contracts, and a ready‑to‑use mock data set.

---

### Goals & Scope

- At‑a‑glance device **flow by Department** with drill‑down to sub‑departments, storage areas, and workstations.
- **Streams** as a first‑class lens (multi‑select cohorts) with **Compare Streams** mode.
- **Path visibility** on the table (coverage per row; highlight path rows only).
- **Δ vs last week** indicators on each metric.
- **Associate lens** (drawer): leaderboard + compare.
- Persist **user prefs** (open rows, columns, deltas, streams, highlight state).

Non‑goals: auth, server calls, real‑time data; everything is local mock.

---

### Tech & Project Structure (suggested)

- **Stack:** React + TypeScript + Vite (or Next) + CSS (Tailwind or CSS Modules). No external charting lib required.
- **Structure:**
  - `src/App.tsx` — page composition
  - `src/components/` — table, chips, column chooser, drawer, etc.
  - `src/data/mock-data.ts` — loads JSON and exposes helpers to derive deltas/coverage
  - `public/mock/pxw-mock-data.json` — the dataset (provided below)
  - `src/styles/tokens.css` — CSS variables

Runbook: `npm create vite@latest pxw-overview -- --template react-ts` → copy files.

---

### Visual System (tokens)

```css
:root{
  --bg:#0f1220; --panel:#151a2e; --panel-2:#101425; --text:#e7ebf6; --muted:#8b93a7;
  --accent:#5dd0ff; --good:#33d17a; --danger:#ff6b6b; --warn:#ffd166; --border:1px solid rgba(255,255,255,.06);
  --row-hover:#1b2140; --badge:#30395f; --chip:#222842; --shadow:0 10px 30px rgba(0,0,0,.35);
  --streamA:#79c0ff; --streamB:#ff9b8a; --streamC:#b287ff;
}
```

Typography: Inter (system fallbacks ok). Use tabular numerals for metric cells.

Layout: Header (sticky) → Subheader (filters) → Table panel → Footer notes. Right‑hand drawer slides over content.

---

### Page Chrome & Controls

- **Header**
  - Title: *Supervisor Overview* (badges: *Demo • Mock Data*, *Facility TZ: Local*)
  - Right: Date picker (default **today**), buttons: *Expand All*, *Collapse All*, *Toggle Annotations*
- **Subheader**
  - **Streams chips** (multi‑select): *All Devices (system)*, *Happy Path*, *Requires Unlock*, *External Polish Overflow*.
  - **Compare Streams** toggle (enabled when ≥2 streams selected)
  - **+ New Stream** (demo: creates a random rule and color)
  - **Column Chooser** popover (Columns + Δ vs LW + Path Coverage)
  - **Highlight Stream Paths** toggle; when on, a floating bar shows *Only show path rows* and *Clear*

---

### Table

Columns (default visible): **Name**, **In**, **WIP**, **Avg WIP**, **Out**, **Exit Rate**, **Lead**. Optional: **Wait**, **Work**, **Path Coverage**.

- **Pinned row:** *Averages & Totals* (sums for counts; out‑weighted averages for time metrics; 8 work hours/day by default)
- **Hierarchy:** Department → Sub‑Dept → Storage / Workstation (chevrons open/close)
- **Row menu (⋯):** Scope toggle (*All Devices* ↔ *Worked‑on only*), *View Associates* (opens drawer), *Highlight paths from here*
- **Δ vs last week:** small chip under each metric; **counts:** green when ▲ up; **time metrics:** green when ▼ down
- **Path Coverage:** either a single bar (primary stream) or stacked 3‑bar when comparing A/B/C streams

**Metric formulas** (per row):

- `Exit Rate (units/hour) = Out / WorkHours`
- `Lead Time (minutes) = AvgWIP / ExitRate * 60`  (Little’s Law proxy)
- `Wait` & `Work` are average minutes for items exiting in the selected period

---

### Associates Drawer

Tabs: **Overview**, **Associates**, **Paths**.

- **Overview:** KPIs: total Out, total Exit Rate, Avg Work Time across visible associates at the selected node
- **Associates:** leaderboard table with *Select to compare* (shows mini KPI cards for selected associates)
- **Paths:** static explanatory text in the mock (no charts required)

Associate metrics per person: `hours`, `out`, (derived) `exitRate`, `avgWork` (min), `p90Work` (min), `waitToStart` (min), `fpy` (0..1), `util` (0..1).

---

### Interactions (acceptance)

- **Hierarchy**
  - Expand/collapse per row. *Expand All / Collapse All* affect all.
- **Per‑row Scope**
  - Default *All Devices*. Toggle to *Worked‑on only* applies to the row + subtree. Persist per user (localStorage key `pxw.overview.v2.prefs`).
- **Search (optional stretch)**
  - Box/IMEI focus behavior out of scope for this mock — ok to omit.
- **Δ vs LW**
  - Enabled/disabled via Column Chooser. Time metrics show green when lower than LW; counts show green when higher.
- **Path Visibility**
  - Column Chooser toggle shows **Path Coverage** column.
  - *Highlight Stream Paths* dims non‑path rows (and optionally hides them when *Only show path rows* is set).
- **Associates**
  - Row menu → *View Associates* opens drawer with the node context. Compare up to 4 associates.
- **Persistence**
  - Save: selected streams, compare mode, open rows, column visibility, Δ on/off, coverage on/off, highlight state.

---

### Data Model (TypeScript)

```ts
export type NodeType = 'Department'|'Sub-Department'|'Storage'|'Workstation';
export interface Node { id:string; name:string; type:NodeType; parent?:string|null; children?:string[]; }
export interface Metrics { in:number; wip:number; avgWip:number; out:number; workTime:number; waitTime:number; hours:number; }
export interface Stream { id:string; name:string; color:string; system?:boolean; }
export interface CoverageMap { [nodeId:string]: number } // share 0..1
export interface Associate { id:string; name:string; role:string; hours:number; out:number; avgWork:number; p90Work:number; waitToStart:number; fpy:number; util:number; }
export interface AssociatesByNode { [nodeId:string]: Associate[] }
export interface PxwDataset {
  date:string; timezone:string;
  streams:Stream[];
  tree:Node[];
  metrics:{ today:{[nodeId:string]:Metrics}; lastWeek?:{[nodeId:string]:Metrics} };
  coverage:{ [streamId:string]: CoverageMap };
  associates:AssociatesByNode;
}
```

**Δ vs LW derivation** (if `metrics.lastWeek` is missing): generate from today using 0.88–1.12 random multipliers per field, preserving hours.

```ts
export function deriveLastWeek(today: {[id:string]:Metrics}): {[id:string]:Metrics} {
  const scale = (n:number, lo=0.88, hi=1.12)=> Math.round(n * (lo + Math.random()*(hi-lo)));
  const out: any = {};
  Object.entries(today).forEach(([id,m])=>{
    out[id] = {
      in: scale(m.in), wip: Math.max(0, Math.round(m.wip * (0.8 + Math.random()*0.4))),
      avgWip: Math.max(0, m.avgWip * (0.85 + Math.random()*0.3)), out: scale(m.out),
      workTime: m.workTime * (0.85 + Math.random()*0.3), waitTime: m.waitTime * (0.85 + Math.random()*0.3),
      hours: m.hours
    } as Metrics;
  });
  return out;
}
```

---

### Components & Contracts

- `<StreamsChips streams selected onSelect compare onToggleCompare />`
- `<ColumnChooser state onChange />` { cols: {in,wip,avgWip,out,exitRate,lead,wait,work}, delta:{enabled}, coverage:{enabled} }
- `<Table rows colState highlight onRowMenu />` → `onRowMenu` returns actions
- `<AssociatesDrawer open node associates streamChips onClose />`
- `<AnnotationPanel enabled onToggle />`

Row shape (for the table renderer):

```ts
interface Row { id:string; name:string; type:NodeType; level:number; pinned?:boolean; scope:'all'|'worked';
  in:number; wip:number; avgWip:number; out:number; exitRate:number; lead:number|null; wait:number; work:number;
  inLW?:number; wipLW?:number; avgWipLW?:number; outLW?:number; exitRateLW?:number; leadLW?:number|null; waitLW?:number; workLW?:number;
  coverage?:{ a:number; b:number; c:number } // happy/unlock/polish in this demo
}
```

---

### Mock Data (JSON)

A ready JSON lives at `public/mock/pxw-mock-data.json`. **Today** metrics are provided; generate **lastWeek** via helper above.

```json
{
  "date": "2025-08-18",
  "timezone": "America/Toronto",
  "streams": [
    {"id":"all","name":"All Devices","color":"#cbd5ff","system":true},
    {"id":"happy","name":"Happy Path","color":"#79c0ff"},
    {"id":"unlock","name":"Requires Unlock","color":"#ff9b8a"},
    {"id":"polish","name":"External Polish Overflow","color":"#b287ff"}
  ],
  "tree": [
    {"id":"receiving","name":"Receiving","type":"Department","parent":null,"children":["dock_storage","check-in"]},
    {"id":"dock_storage","name":"Dock Storage","type":"Storage","parent":"receiving","children":[]},
    {"id":"check-in","name":"Check-in","type":"Workstation","parent":"receiving","children":[]},

    {"id":"testing","name":"Testing","type":"Department","parent":null,"children":["functional_test_a","functional_test_b","exceptions_store"]},
    {"id":"functional_test_a","name":"Functional Test A","type":"Workstation","parent":"testing","children":[]},
    {"id":"functional_test_b","name":"Functional Test B","type":"Workstation","parent":"testing","children":[]},
    {"id":"exceptions_store","name":"Exceptions Store","type":"Storage","parent":"testing","children":[]},

    {"id":"unlock","name":"Unlock","type":"Department","parent":null,"children":["unlock_bench_1","unlock_bench_2"]},
    {"id":"unlock_bench_1","name":"Unlock Bench 1","type":"Workstation","parent":"unlock","children":[]},
    {"id":"unlock_bench_2","name":"Unlock Bench 2","type":"Workstation","parent":"unlock","children":[]},

    {"id":"external_polish","name":"External Polish","type":"Department","parent":null,"children":["polish_queue","polish_station_1"]},
    {"id":"polish_queue","name":"Polish Queue","type":"Storage","parent":"external_polish","children":[]},
    {"id":"polish_station_1","name":"Polish Station 1","type":"Workstation","parent":"external_polish","children":[]},

    {"id":"qc","name":"QC","type":"Department","parent":null,"children":["qc_bench_1"]},
    {"id":"qc_bench_1","name":"QC Bench 1","type":"Workstation","parent":"qc","children":[]},

    {"id":"grading","name":"Grading","type":"Department","parent":null,"children":["grading_bench_1"]},
    {"id":"grading_bench_1","name":"Grading Bench 1","type":"Workstation","parent":"grading","children":[]},

    {"id":"sort_hub","name":"Sort Hub","type":"Department","parent":null,"children":["outbound"]},
    {"id":"outbound","name":"Outbound","type":"Storage","parent":"sort_hub","children":[]}
  ],
  "metrics": {
    "today": {
      "receiving": {"in":520,"wip":10,"avgWip":18,"out":500,"workTime":12,"waitTime":22,"hours":8},
      "dock_storage": {"in":320,"wip":4,"avgWip":6,"out":300,"workTime":5,"waitTime":35,"hours":8},
      "check-in": {"in":200,"wip":6,"avgWip":12,"out":200,"workTime":18,"waitTime":15,"hours":8},

      "testing": {"in":500,"wip":14,"avgWip":24,"out":480,"workTime":30,"waitTime":28,"hours":8},
      "functional_test_a": {"in":250,"wip":7,"avgWip":12,"out":240,"workTime":28,"waitTime":26,"hours":8},
      "functional_test_b": {"in":240,"wip":6,"avgWip":10,"out":230,"workTime":30,"waitTime":27,"hours":8},
      "exceptions_store": {"in":30,"wip":1,"avgWip":2,"out":10,"workTime":5,"waitTime":60,"hours":8},

      "unlock": {"in":140,"wip":5,"avgWip":8,"out":130,"workTime":25,"waitTime":35,"hours":8},
      "unlock_bench_1": {"in":70,"wip":2,"avgWip":4,"out":65,"workTime":25,"waitTime":28,"hours":8},
      "unlock_bench_2": {"in":70,"wip":3,"avgWip":4,"out":65,"workTime":26,"waitTime":30,"hours":8},

      "external_polish": {"in":110,"wip":3,"avgWip":7,"out":100,"workTime":20,"waitTime":45,"hours":8},
      "polish_queue": {"in":110,"wip":2,"avgWip":4,"out":100,"workTime":5,"waitTime":50,"hours":8},
      "polish_station_1": {"in":100,"wip":1,"avgWip":3,"out":95,"workTime":22,"waitTime":20,"hours":8},

      "qc": {"in":480,"wip":8,"avgWip":15,"out":470,"workTime":14,"waitTime":18,"hours":8},
      "qc_bench_1": {"in":480,"wip":8,"avgWip":15,"out":470,"workTime":14,"waitTime":18,"hours":8},

      "grading": {"in":470,"wip":6,"avgWip":12,"out":460,"workTime":20,"waitTime":16,"hours":8},
      "grading_bench_1": {"in":470,"wip":6,"avgWip":12,"out":460,"workTime":20,"waitTime":16,"hours":8},

      "sort_hub": {"in":460,"wip":5,"avgWip":9,"out":455,"workTime":8,"waitTime":12,"hours":8},
      "outbound": {"in":455,"wip":0,"avgWip":2,"out":455,"workTime":4,"waitTime":5,"hours":8}
    }
  },
  "coverage": {
    "happy": {
      "receiving":1.0,"dock_storage":0.65,"check-in":0.35,
      "testing":0.82,"functional_test_a":0.50,"functional_test_b":0.48,"exceptions_store":0.04,
      "unlock":0.00,"unlock_bench_1":0.00,"unlock_bench_2":0.00,
      "external_polish":0.10,"polish_queue":0.08,"polish_station_1":0.07,
      "qc":0.76,"qc_bench_1":0.76,
      "grading":0.73,"grading_bench_1":0.73,
      "sort_hub":0.71,"outbound":0.71
    },
    "unlock": {
      "receiving":1.0,"dock_storage":0.40,"check-in":0.25,
      "testing":0.62,"functional_test_a":0.36,"functional_test_b":0.34,"exceptions_store":0.06,
      "unlock":0.41,"unlock_bench_1":0.22,"unlock_bench_2":0.19,
      "external_polish":0.04,"polish_queue":0.04,"polish_station_1":0.03,
      "qc":0.35,"qc_bench_1":0.35,
      "grading":0.31,"grading_bench_1":0.31,
      "sort_hub":0.29,"outbound":0.29
    },
    "polish": {
      "receiving":1.0,"dock_storage":0.42,"check-in":0.30,
      "testing":0.55,"functional_test_a":0.30,"functional_test_b":0.28,"exceptions_store":0.03,
      "unlock":0.00,"unlock_bench_1":0.00,"unlock_bench_2":0.00,
      "external_polish":0.22,"polish_queue":0.22,"polish_station_1":0.20,
      "qc":0.18,"qc_bench_1":0.18,
      "grading":0.16,"grading_bench_1":0.16,
      "sort_hub":0.15,"outbound":0.15
    }
  },
  "associates": {
    "check-in": [
      {"id":"u_li","name":"Li Zhang","role":"Receiving","hours":6.2,"out":120,"avgWork":17,"p90Work":27,"waitToStart":10,"fpy":0.94,"util":0.78},
      {"id":"u_bob","name":"Bob Nguyen","role":"Tester","hours":5.8,"out":80,"avgWork":20,"p90Work":32,"waitToStart":12,"fpy":0.92,"util":0.72}
    ],
    "functional_test_a": [
      {"id":"u_bob","name":"Bob Nguyen","role":"Tester","hours":6.5,"out":130,"avgWork":28,"p90Work":45,"waitToStart":14,"fpy":0.93,"util":0.81}
    ],
    "functional_test_b": [
      {"id":"u_bob","name":"Bob Nguyen","role":"Tester","hours":6.1,"out":110,"avgWork":30,"p90Work":48,"waitToStart":16,"fpy":0.91,"util":0.77}
    ],
    "unlock_bench_1": [
      {"id":"u_asha","name":"Asha Patel","role":"Unlock","hours":6.8,"out":70,"avgWork":26,"p90Work":42,"waitToStart":18,"fpy":0.90,"util":0.74}
    ],
    "unlock_bench_2": [
      {"id":"u_asha","name":"Asha Patel","role":"Unlock","hours":6.4,"out":65,"avgWork":27,"p90Work":44,"waitToStart":19,"fpy":0.89,"util":0.72}
    ],
    "polish_station_1": [
      {"id":"u_ken","name":"Ken Park","role":"Polish","hours":6.0,"out":95,"avgWork":22,"p90Work":36,"waitToStart":20,"fpy":0.95,"util":0.80}
    ],
    "qc_bench_1": [
      {"id":"u_miguel","name":"Miguel Santos","role":"QC","hours":6.7,"out":170,"avgWork":14,"p90Work":24,"waitToStart":12,"fpy":0.96,"util":0.82}
    ],
    "grading_bench_1": [
      {"id":"u_sally","name":"Sally Ortega","role":"Grader","hours":6.3,"out":160,"avgWork":20,"p90Work":32,"waitToStart":14,"fpy":0.94,"util":0.79}
    ]
  }
}
```

> Note: Hours assume an 8‑hour workday. Lead/ExitRate are UI‑derived. Coverage values are 0..1 shares of stream devices that *touched* each node.

---

### Implementation Notes

- Keep numbers deterministic during a session by seeding `Math.random` if you add any local jitter.
- Use `localStorage` key `pxw.overview.v2.prefs` to persist UI state.
- Time formatting: `HHh MMm` (e.g., `04h 27m`).
- Weights for *Averages & Totals*: out‑weighted for time metrics, sum for counts.
- Path highlight threshold: consider a row “on path” if coverage for the primary stream ≥ **0.2**.

---

### Accessibility & UX

- Focus order should include chips, Column Chooser button, highlight toggle, table chevrons, row menus, drawer close.
- Buttons and toggles require visible focus outlines.
- Announce drawer open/close via `aria-live` (optional for mock).

---

### Out of Scope / Future

- Device‑level search/focus, box lookup, cross‑day ranges, real back‑end.

---

*Spec + mock data prepared for Cursor integration.*

