# Site Planner — Build Plan

## What We're Building (Simple Terms)

A digital site-planning tool — think Google Maps meets a drawing canvas. The user loads a floor plan or satellite map, places cameras, cores, and add-on devices onto it, and the tool automatically draws colored cones showing exactly what each camera can see (and what it misses). As devices are added, a live shopping list (BOM) builds itself on the side with prices, licensing, and storage estimates. The final output is a professional PDF report the user can send to a client.

It is equivalent to tools like IPVM's Camera Calculator or Genetec Site Designer, but built into Lumana.

> **Device types supported:** Video cameras, Lumana Cores (recording appliances), and Add-on devices (per-camera: Cloud Storage, External NAS, Timelapse, Cloud Metadata, Cloud Snapshot; per-plan: Alerts Storage, Event Tags Storage, Video Verification). Access control devices (door controllers, readers) and sensors (air quality, intercoms) are planned for a future phase.

---

## 0. Application Pages & Navigation Flow

The Site Planner is a multi-page application. Each page is a standalone HTML prototype (inline CSS + JS, no build tools). Pages link to each other via URL parameters.

```
site-planner-sites.html              (Projects list — entry point)
    ↓  click project row
site-planner-location.html           (Project hub — locations, BOM, project details)
    ├→  sidebar click location        ?siteId=X&locationId=Y
    ├→  sidebar click plan            ?siteId=X&locationId=Y&planId=Z
    ├→  "View Plan" button
    │   site-planner-prototype.html   (Full interactive floor-plan editor)
    └→  read-only link
        site-planner-plan.html        (Plan viewer — BOM only, no canvas)
```

---

### site-planner-sites.html — Projects List (~1,110 lines)

Landing page showing all projects in a filterable, searchable table.

| Element | Details |
|---|---|
| **Layout** | 64px left nav + full-width main area |
| **Top bar** | Breadcrumb ("Projects") + "+ New Project" button |
| **Tabs** | My projects · Shared with me · Closed (with search + date-range + win/loss filters; win/loss filter only on Closed tab) |
| **Project table** | Columns: Project name (purple link) · BOM (device count) · Created date · Actions (⋯ menu: Edit, Duplicate, Close/Reopen, Delete). Closed tab adds a Closure reason column. |
| **Inline row actions** | Click row → navigates to `site-planner-location.html?siteId=X` |
| **Create Project modal** | Project name + optional contacts (add/remove rows) + unit selection (Imperial/Metric) + notes |
| **Create Location modal** | Name + address input with draggable Leaflet marker (optional) |
| **Create Plan modal** | Plan name + optional floor-plan image upload (PDF/JPG/PNG ≤15MB, with preview) |
| **Close Reason modal** | Win/Loss toggle |
| **Floor Plan Wizard** | 3-step overlay (Crop → Scale → Align) triggered after uploading a floor plan |
| **Data** | `SITES` (id, name, status, createdDate, closureReason, bom), `LOCATIONS`, `PLANS`, `DEVICES` arrays |
| **State** | `sites` mutable array, `activeTab`, `searchQuery`, `statusFilter`, `openDropdown` |

---

### site-planner-location.html — Project Hub (~2,460 lines)

The central hub between the Projects list and the interactive Plan editor. Three distinct view modes based on URL parameters. All three views share a single `renderAllLocationsBom()` function that scopes data based on the active view.

**Shared layout:** 64px left nav → 220px sidebar (location/plan tree, always shows full hierarchy) → main content area with top bar breadcrumb + action buttons.

#### Top Bar Actions (per view)

| View | Buttons |
|---|---|
| **All Locations** | Project details (edit icon) · \| · Close/Reopen · Share · Export |
| **Single Location** | Delete (icon) · + Create plan |
| **Single Plan** | Delete (icon) |

> The project-wide Delete action lives inside the Project Details modal (not on the top bar). Share and Export are currently placeholder alerts. Creating a new location happens from inside the sidebar tree, not from a top-bar button.

#### View Mode 1: All Locations (no `locationId` or `planId`)

The most feature-rich view — project-wide BOM management.

| Element | Details |
|---|---|
| **Project Details** | Edit icon in topbar opens a modal in edit mode (single column, 20px gaps). Fields: project name, created date, owner, status badge, unit system, shared users, contacts, notes. Delete button lives at the bottom of this modal. |
| **View Options** | Always-visible panel: search input, device type filter dropdown (dynamic from data), group-by toggle (**Cores / Device type**), sort (A-Z, Price ↑, Price ↓) |
| **Bulk Editing Toolbar** | Items All/Selected toggle with select-all checkbox, Upfront/Yearly payment, 1/3/5/10yr duration, Retention dropdown, License dropdown, +Add-ons button, Delete button, Expand/Collapse all (right-aligned) |
| **BOM Table** | Hierarchical collapsible: Location (purple left border, #F8FAFC bg, 16px semibold — always shown, even in Single Location view) → Plan → Core group (colored left border, chevron-only expand, name click opens device info modal) → Cameras (click opens modal). Alternate "Device type" flat grouping. |
| **Grand Total** | Label in dark grey, amount in 24px bold purple, tabular-nums font |
| **What's Included** | Collapsible link below Grand Total, right-aligned, 670px max width, lilac background (#faf8ff) with purple border. 10 features in 2-column grid. |

#### View Mode 2: Single Location (`locationId` set, no `planId`)

| Element | Details |
|---|---|
| **Content header** | Location address with pin icon |
| **BOM** | Same shared BOM table (search, filters, toolbar) scoped to this location's plans. Group-by toggle shows **Cores / Device type**. Location header row is still rendered above the plans. |

#### View Mode 3: Single Plan (`planId` set)

| Element | Details |
|---|---|
| **Map preview (280px)** | Leaflet map with floor plan overlay SVG + "View Plan" button → links to `site-planner-prototype.html` |
| **Content header** | Location address with pin icon |
| **BOM toolbar** | Same shared search, filters, and Items toolbar as All Locations view |
| **Hardware category** | Collapsible `.bom-category` block with header (title, device count badge, total), body (core groups → cameras, same `.all-bom-core-group` markup), footer ("Hardware Total"). `hwTotal = computePlanBomTotal(planId)`. |
| **Software category** | Collapsible `.bom-category` block. Body: single license row ("Lumana License", N × $199). Footer: "Software Total". Only shown if cameras exist. |
| **Grand Total** | Hardware Total + Software Total. Same 24px purple styling. |
| **What's Included** | Same collapsible block as All Locations view. |

#### Shared BOM Rendering (`renderAllLocationsBom()`)

One function renders all three views. Key scoping logic:
```
bomLocations = showPlan ? [planLocation] : (!showAllLocations ? [currentLocation] : siteLocations)
allPlanIds   = showPlan ? [planId] : PLANS.filter(p => allLocIds.includes(p.locationId)).map(p => p.id)
```
Plan tab uses a dedicated `if (showPlan)` branch that renders Hardware/Software category blocks instead of the groupBy locations/type table.

#### Modals

| Modal | Details |
|---|---|
| **Project Details** | Edit mode only (opens from topbar edit icon). Single column layout, 20px gaps. Project name, date, owner, status, unit, shared users, contacts, notes. Cancel/Save + Delete. |
| **Device Info** (Camera) | Name (editable), Model, Type, Color picker (8 colors), Price, Core assignment dropdown. Cancel/Save. |
| **Device Info** (Core) | Name (editable), Type, Channels, Color picker, Payment (Upfront/Yearly), Duration (1/3/5/10yr, conditional on Yearly), Price, Assigned Cameras list with tags (payment, duration, license, retention) and per-camera price. Cancel/Save. |
| **Device Info** (Device) | Model, Category, Qty, Accessories, Unit price, Total (read-only). Cancel/Save. |
| **Delete Confirmation** | Red trash icon, item count summary, Cancel/Delete. |
| **Close Project** | Win/Loss toggle, optional notes, Cancel/Close. |
| **Create Location** | Location name + address inputs, map preview. |
| **Create Plan** | Breadcrumb context, plan name input, floor plan upload area. |

#### Data Model

| Array | Key Fields |
|---|---|
| `SITES` | id, name, status, createdDate, owner, closureReason, closureNotes, unit, contacts[], sharedWith[], notes |
| `LOCATIONS` | id, siteId, name, address |
| `PLANS` | id, locationId, name |
| `DEVICES` | planId, category, model, qty, accessories, price |
| `CORES` | planId, name, type, channels, cameraCount, price, color |
| `PLAN_CAMERAS` | planId, name, model, type, price, color, coreIdx (synthetic `_id` at init) |

Constants: `LICENSE_PRICE = 199` (per camera, for software licensing).

Hierarchy: Site → Locations → Plans → (Cores → Cameras + Devices)

#### State Objects

`_allBomViewOptions` (search, device filters, groupBy, sort), `_allBomScope` (mode, selectedCameras Set, selectedCores Set, bulk edit values), `_allBomCollapse` (expand/collapse state keyed by loc/plan/core/type), `_bomCategoryCollapse` (hardware/software collapse state for plan tab), `_allBomAllExpanded`, `_allBomIncludedOpen`, `_deviceInfoContext`.

---

### site-planner-plan.html — Plan Viewer (456 lines)

Read-only view of a single plan. Same sidebar + data structure as `site-planner-location.html` but without the interactive canvas.

| Element | Details |
|---|---|
| **Layout** | Nav + sidebar + main content area |
| **Map preview** | 280px Leaflet map with optional floor-plan overlay SVG and a "View plan" button → links to `site-planner-prototype.html` |
| **Content** | Content header (address, device count), project details card, Hardware/Software collapsible BOM sections with device rows |
| **Top bar** | Edit and Export buttons (currently placeholder alerts) |
| **Use case** | Stakeholder review — no editing capability |

---

### site-planner-prototype.html — Interactive Editor (~6,250 lines)

The main floor-plan editor. Full canvas-based editor built on Konva + Leaflet + Three.js. Covered in detail in the rest of this document.

---

## I. Recommended Tech Stack

### For the Interactive Prototype (this repo's style — standalone HTML)
| Layer | Choice | Why |
|---|---|---|
| Canvas / drawing | **Konva.js** (CDN) | Best 2D canvas lib for draggable, interactive objects. Handles FoV cones, walls, snap-to-grid natively. |
| Maps | **Leaflet.js** (CDN) + OpenStreetMap | Free, no API key needed for prototype. Swap to Google Maps API in production. |
| 3D preview | **Three.js** (CDN) | Industry standard WebGL. Lightweight enough for an in-browser preview pane. |
| PDF export | **jsPDF + html2canvas** (CDN) | Capture canvas + BOM table into a shareable PDF. |
| URL sharing | **LZString** (CDN) | Compresses plan JSON into URL hash for serverless sharing. |
| State | Vanilla JS (plain objects + event listeners) | Sufficient for prototype scope. |

### For Production (engineering handoff recommendation)
| Layer | Choice |
|---|---|
| Framework | **React 18** + TypeScript |
| Canvas | **Konva.js** via `react-konva` |
| Maps | **Google Maps JS API** + `@react-google-maps/api` |
| 3D | **Three.js** via `@react-three/fiber` + `@react-three/drei` |
| State | **Zustand** (lightweight, single store) |
| PDF | **@react-pdf/renderer** or Puppeteer (server-side) |
| File parsing | **pdf.js** (floor plan PDF upload), **dxf-parser** (CAD/DWG) |
| Backend sync | REST API calls to existing Lumana backend |

---

## II. File Structure

```
Lumana-Desktop/
├── site-planner-sites.html              ← Projects list (entry point, ~1,110 lines)
├── site-planner-location.html           ← Project hub + BOM management (~2,460 lines)
├── site-planner-plan.html               ← Read-only plan viewer (~456 lines)
├── site-planner-prototype.html          ← Full interactive floor-plan editor (~6,250 lines)
│
├── Knowledge/
│   ├── SitePlanner-Plan.md              ← This file (build plan & architecture)
│   └── SitePlanner-Locations.md         ← Detailed location page documentation
│
├── figma-assets/                        ← SVG icons for Figma / design use
│   ├── dome-camera.svg
│   ├── bullet-camera.svg
│   ├── fisheye-camera.svg
│   ├── camera-generic.svg
│   ├── core-icon.svg
│   ├── bom-icon.svg
│   ├── floor-plan-icon.svg
│   ├── site-planner-icon.svg
│   └── site-planner-icon-v2.svg
│
└── assets/
    └── site-planner/
        ├── camera-dome.svg
        ├── camera-bullet.svg
        ├── camera-fisheye.svg
        └── camera-multisensor.svg
```

---

## III. Design Considerations

### Visual Language
- **Match existing Lumana UI**: dark sidebar, white canvas area, purple accent (`#7c3aed`).
- **Canvas background**: dark satellite-style (`#1c2333`) with dot grid pattern.
- **Camera icons**: small colored circle markers on canvas with camera-type symbol.
- **Core icons**: rounded rectangle with circuit-board SVG, colored by user choice.
- **Add-on icons**: rounded rectangle with storage icon (horizontal lines), colored by type.
- **FoV cones**: single gradient cone using the camera's brand color — **no multi-color DORI zones**. One cone with three opacity layers of the same hue: Detection (outer) ~15%, Recognition (middle) ~30%, Identification (inner) ~50%.
- **Walls/obstructions**: thick lines; red solid / blue dashed (glass) / amber dotted (fence).

### Layout (2-panel — toolbar + canvas + right panel)
```
┌─────────────────────────────────────────────────────┐
│  Top bar: breadcrumb (Project > Location > Plan)    │
│  Action bar: Share, Export PDF, Export CSV, Save     │
├─────────────────────────────────────────────────────┤
│  Toolbar: +Device, +Camera | Select, Move, Wall |   │
│           Basemap | Zoom | Clear All                │
├──────────────────────────────────┬──────────────────┤
│                                  │                  │
│     Map Canvas (center)          │  Properties      │
│     [Konva stage over Leaflet]   │  Panel (700px)   │
│                                  │  + BOM tab       │
│                                  │                  │
└──────────────────────────────────┴──────────────────┘
```

> **Note:** The original left catalog panel was removed. Device creation is now handled entirely from the toolbar via `+ Device` dropdown (Core / Add-On) and `+ Camera` button.

### Device Placement UX

**Cameras:**
- `+ Camera` toolbar button opens Camera Model Picker modal (IPVM-style 3-column browser)
- Search by brand, model, form factor, resolution, IR range, NDAA compliance
- `CAMERA_DB` — 40+ models across 8 brands (Axis, Hanwha, Bosch, Hikvision, Dahua, Avigilon, Pelco, Generic)
- Placing pre-fills FoV angle, DORI ranges, focal length, color

**Cores:**
- `+ Device → Core` opens Core Type Picker modal
- 4 types: Core 10 (10ch/$1,500), 1U Core 64 (64ch/$5,000), 2U Core 128 (128ch/$9,000), 2U Core 256 (256ch/$15,000)
- Click to place on canvas, enter name, view properties

**Add-Ons:**
- `+ Device → Add-On` opens Add-On Picker modal
- Scope toggle: **Per camera** (5 types) | **Per plan** (3 types)
- Per-camera types: Cloud Storage ($500), External NAS Storage ($800), Timelapse ($200), Cloud Camera Metadata ($150), Cloud Camera Snapshot ($100) — assigned to specific cameras, some with retention options
- Per-plan types: Alerts Storage ($300), Event Tags Storage ($250), Video Verification ($400) — payment (Upfront/Yearly) + duration options
- Camera selection modal with search when placing per-camera add-ons
- Also accessible from camera properties panel → "Add add-on" button (skips scope toggle, forces per-camera)

### Interaction Principles
1. **Click to select** a device → right panel shows its configuration.
2. **`+ Camera` button** → opens picker modal → select model → "Place Camera" → click canvas to place.
3. **`+ Device` dropdown** → Core or Add-On → opens respective picker → click canvas to place.
4. **Rotate handle** on selected camera → cone rotates.
5. **Wall tool** → click-click to draw an obstruction line that clips cones.
6. **Scale tool** → draw a line, type real-world length → sets pixels-per-meter ratio.
7. **Right-click** on device → Delete / Duplicate / Properties.
8. **Keyboard shortcuts**: `V` select, `H` move, `C` camera, `W` wall, `Del` delete, `⌘Z`/`⌘⇧Z` undo/redo, arrows nudge.

### DORI Zone Logic (FoV cone math)
For a camera at mounting height `H`, tilt angle `θ`, and focal length `f`:
- **Detection range** = sensor_width / (focal_length_mm × 0.25 px/m) *(simplified)*
- Each DORI zone is a nested arc segment within the main cone.
- Cone clips against wall segments using line-segment intersection (ray casting).
- **Glass walls**: ray continues past the wall at 55% of remaining range.
- **Fence/barriers**: ray continues at 35% of remaining range.

### Device Relationships
- **Camera → Core**: a camera can be assigned to one core; a core holds many cameras (up to its channel capacity). Tracked via `cam.coreId`.
- **Per-camera Add-On → Camera**: an add-on with `scope:'camera'` has a `cameraId` field (one-to-one from add-on side). A camera can have multiple add-ons assigned (one-to-many from camera side). Found via `state.addons.filter(a => a.cameraId === camId)`.
- **Per-plan Add-On**: `scope:'plan'` add-ons have `cameraId: null` — they belong to the project, not individual cameras. Found via `state.addons.filter(a => ADDON_TYPES[a.typeId].scope === 'plan')`.
- The camera properties panel shows "Core Assignment" and "Assigned add-ons" (per-camera) blocks.
- The BOM tab shows per-camera add-ons nested under their camera, per-plan add-ons in a separate "Plan Add-Ons" group.

---

## IV. Step-by-Step Implementation Plan

### Phase 1 — Canvas Foundation ✅ COMPLETE

1. **[✅] HTML shell** — Three-panel layout, Lumana nav sidebar, top bar, toolbar, status bar
2. **[✅] Map Canvas** — Konva stage, demo floor plan with rooms + furniture, zoom/pan, grid toggle
3. **[✅] Camera Placement** — Click-to-place, rotation handle drag, arrow key nudge
4. **[✅] FoV Cone Rendering** — Ray-cast DORI zones (Detection/Recognition/Identification), clips against walls in real-time
5. **[✅] Scale Calibration Tool** — Draw line → enter real-world length → sets px/m ratio
6. **[✅] Wall Drawing Tool** — Click-click to draw walls; cones clip immediately
7. **[✅] Properties Panel** — DORI zone toggles, resolution, FoV angle, mount height, tilt, retention, license; live dead zone / storage / bandwidth calc
8. **[✅] Live BOM** — Auto-updating table with unit + total MSRP
9. **[✅] Floor Plan Upload** — JPG/PNG/PDF replaces demo floor plan on canvas
10. **[✅] Undo/Redo** — Full stack (⌘Z / ⌘⇧Z)
11. **[✅] Context Menu** — Right-click: Duplicate / Properties / Delete

---

### Phase 2 — Camera Model Picker ✅ COMPLETE

12. **[✅] Camera Model Picker Modal** *(IPVM-style 3-column browser)*
    - **Filter bar:** search input + form-factor chips (All / Dome / Bullet / Fisheye / Multi-sensor / PTZ) + NDAA-only toggle + resolution/IR/HAoV sliders
    - **3-column body:**
      - Left 180px: scrollable brand list with live model counts
      - Middle 300px: model list with form factor icon, model name, NDAA badge, price
      - Right flex: info panel — model name, brand, resolution/form-factor badges, FoV cone SVG preview, 6-spec grid, MSRP
    - **Footer:** live model count, Cancel, **Place Camera** CTA
    - `CAMERA_DB` — 40+ models across 8 brands with DORI ranges, NDAA flags, form factors

---

### Phase 3 — Gradient FoV Cone Visualization ✅ COMPLETE

13. **[✅] Replace multi-color DORI polygons with single-color gradient layers**
    - Each camera has one `color` field; all three zone layers use it
    - Konva `opacity` per layer: `idR` = 0.50, `recR` = 0.30, `detR` = 0.15
    - Subtle stroke outline (same color, 0.4 opacity) at outer detection boundary

---

### Phase 4 — Wall Types & Obstruction Modeling ✅ COMPLETE

14. **[✅] Wall Type Selector**
    - Sub-toolbar when Wall tool active: Solid · Glass · Fence
    - **Solid:** full FoV block | **Glass:** 55% attenuation | **Fence:** 35% attenuation
    - Visual distinction: solid red / blue dashed / amber dotted
    - Canvas legend appears when user walls are present

---

### Phase 5 — Camera Configuration ✅ COMPLETE

15. **[✅] Focal Length → FoV binding**
    - Focal length slider (1.8–25mm) with preset chips: 2.8 / 4 / 6 / 8 / 12 / 25mm
    - Formula: `FoV = 2 × atan(3.2 / focalMm) × 180/π` (1/2.8" sensor)
    - Bidirectional sync between focal length and FoV angle
    - Fisheye: "Fixed fisheye — 360°" display

---

### Phase 6 — Enhanced BOM Panel ✅ COMPLETE

16. **[✅] Project Total Header**
    - Always-visible top section of right panel: large MSRP total + "estimate" label + purple progress bar

17. **[✅] Tabbed Right Panel: Properties | BOM**
    - Properties tab: device configuration
    - BOM tab: full project breakdown with device count badge

18. **[✅] BOM — Grouped by Core**
    - Cameras grouped under assigned cores (core name, channel info, subtotal)
    - Unassigned cameras group
    - Per-camera row: name, model, resolution/retention/license/payment tags (inline-editable)
    - **Bulk edit mode**: select all/individual, batch update retention/license/payment
    - **Drag mode**: drag cameras between cores or to unassigned
    - CSV export with all sections

---

### Phase 7 — Bandwidth & Storage Calculator ✅ COMPLETE

19. **[✅] Per-camera calculations**
    - Bitrate by resolution: 1080p=4Mbps, 4K=16Mbps, 8MP=10Mbps, 12MP=14Mbps
    - Storage: `bitrate × 86400 / 8 / 1024` GB/day × retention days
    - 3-chip display: Mbps · GB/day · TB/retention

20. **[✅] Site-wide summary** *(BOM pane)*
    - Total bandwidth (Mbps), total storage (TB)
    - NVR recommendation via `calcNVR()`: 2TB/4TB/8TB/16TB drive tiers
    - Included in CSV and PDF exports

---

### Phase 8 — Satellite Map & Floor Plan Enhancement ✅ COMPLETE

21. **[✅] Address Search + Satellite Background**
    - Basemap panel toggle in toolbar
    - Mode toggle: Floor Plan Only | Map Only | Floor Plan & Map
    - ESRI World Imagery satellite tiles via Leaflet
    - Address search → Nominatim geocoding → flies map to location
    - Konva stage renders transparently over Leaflet

22. **[✅] Floor Plan Opacity & Editing**
    - Opacity slider (0–100%) in Floor Plan & Map mode
    - Floor plan editor: drag to reposition, corner handles for resize, rotation with snap points
    - Upload/replace/delete floor plan from basemap panel

23. **[✅] Scale Auto-Detection**
    - `syncScaleFromMap()` — derives scalePPM from Leaflet zoom + latitude (Web Mercator formula)
    - Updates on every zoom/pan; status bar shows "(map)" suffix

---

### Phase 9 — 3D Scene Preview ✅ COMPLETE

24. **[✅] 3D Preview Panel**
    - Three.js (r160) WebGL scene: grid floor, mounting pole, camera body with lens
    - Semi-transparent FoV frustum cones (Det/Rec/Id) in camera's brand color
    - Dead-zone red circle at ground level
    - DORI range rings on floor
    - Mouse-drag orbit, scroll zoom

25. **[✅] Mounting Variables in 3D**
    - Mount height and tilt angle update frustum in real-time
    - Target distance slider (0.5–20m) with test person avatar
    - Selection change → 3D refreshes instantly

---

### Phase 10 — PDF Report Export ✅ COMPLETE

26. **[✅] PDF Report Generation**
    - 3-page A4 PDF via jsPDF 2.5
    - **Page 1 — Cover:** dark background, purple hero band, project name + date, 4 stat cards (cameras / MSRP / storage / bandwidth), NVR recommendation
    - **Page 2 — Floor Plan:** Konva `stage.toDataURL()` screenshot (2×), camera dots + numbered labels, legend
    - **Page 3 — BOM:** cameras table + licensing + storage & network + grand total
    - File saved as `{project-name}-site-plan.pdf`

---

### Phase 11 — Save, Share & Collaborate ✅ COMPLETE

27. **[✅] Save / Load Plan (JSON)**
    - `planSnapshot()` — serializes cameras, walls, scale, nextId, projectName, savedAt (v2 schema)
    - "Save" button → downloads `{project-name}.json`
    - "Open" button → file picker → `loadDesign()` restores full state
    - Handles v1 + v2 file formats

28. **[✅] URL Share Link** *(serverless)*
    - "Share" button → share modal with generated link
    - LZString `compressToEncodedURIComponent` → `#plan=...` URL fragment
    - "Copy link" with clipboard feedback
    - Auto-restores plan from hash on page load

---

### Phase 12 — Core Devices ✅ COMPLETE

29. **[✅] Core Type Picker Modal**
    - 4 core types: Core 10 (10ch/$1,500), 1U Core 64 (64ch/$5,000), 2U Core 128 (128ch/$9,000), 2U Core 256 (256ch/$15,000)
    - `CORE_TYPES` constant with id, name, channels, price

30. **[✅] Core Placement & Canvas Rendering**
    - Click to place on canvas → Konva group with colored rounded rect body + circuit-board icon
    - Inline name input after placement
    - Drag to reposition, click to select, selection ring highlight
    - Labels rendered below icon

31. **[✅] Core Properties Panel**
    - Name, type dropdown, color picker (8 colors), channel capacity display
    - Payment method (Upfront/Yearly with duration selector)
    - **Assigned cameras section**: list of cameras assigned to this core with name/model/resolution
    - "Add cameras" button → modal with searchable list of unassigned cameras, multi-select checkboxes
    - Remove camera (×) button per row
    - Click camera name → navigates to camera properties
    - Delete button in panel header

32. **[✅] Camera ↔ Core Assignment**
    - Camera properties panel: "Core Assignment" block with dropdown of available cores
    - "View core" link to navigate to core properties
    - Assignment tracked via `cam.coreId` field
    - BOM groups cameras by assigned core

---

### Phase 13 — Add-On Devices ✅ COMPLETE

33. **[✅] Add-On Picker Modal**
    - **Scope toggle**: "Per camera" | "Per plan" — filters available types
    - Per-camera types (5): Cloud Storage ($500), External NAS Storage ($800), Timelapse ($200), Cloud Camera Metadata ($150), Cloud Camera Snapshot ($100)
    - Per-plan types (3): Alerts Storage ($300), Event Tags Storage ($250), Video Verification ($400)
    - Conditional fields based on type: retention selector (per-camera storage), payment method + duration (per-plan)
    - Camera selection modal with search for per-camera add-ons
    - `ADDON_TYPES` constant (scope, price, color, fields, retentionOpts), `ADDON_ICON` SVG constant

34. **[✅] Add-On to BOM (no canvas placement)**
    - Add-ons are added directly to the BOM — no canvas placement step
    - Per-camera add-ons appear nested under their assigned camera in the BOM
    - Per-plan add-ons appear in a separate "Plan Add-Ons" section in the BOM
    - Add-on properties viewable in right panel on selection

35. **[✅] Add-On Properties Panel** *(simplified — no Name/Color fields)*
    - Type dropdown (filtered by scope) — Name and Color fields were removed; BOM shows the type label only
    - Retention selector (per-camera types with retention options)
    - Payment method + duration (per-plan types)
    - Camera section (per-camera types): shows assigned camera name (clickable → navigates to camera), assign/remove
    - Delete button in panel header

36. **[✅] Add-On Assignment in Camera Properties**
    - "Assigned add-ons" section in camera properties panel
    - "Add add-on" button → opens Add-On Picker in per-camera mode (scope toggle hidden, camera pre-selected)
    - Assigned add-ons displayed as rows: icon, name/type, retention tag, price, remove (×) button
    - Click add-on name → navigates to add-on properties

---

### Phase 14 — Toolbar Restructuring ✅ COMPLETE

37. **[✅] Removed left catalog panel**
    - Entire `<aside class="catalog">` removed along with all catalog CSS and JS functions
    - Canvas area now stretches full width (sidebar to properties panel)

38. **[✅] `+ Device` dropdown button**
    - Toolbar button with arrow-down icon, no icon on left
    - Dropdown with "Core" and "Add-On" options
    - Replaces the old `+ Core` button

39. **[✅] `+ Camera` toolbar button**
    - Purple dominant button in toolbar
    - Opens Camera Model Picker modal directly

---

### Phase 15 — Integrations (Production) ⬜ NOT STARTED

40. **[ ] Salesforce Push** — Map BOM line items to Salesforce Opportunity via API
41. **[ ] Verkada Command Sync** — Push placed cameras + positions to Command platform
42. **[ ] Access Control + Sensors** — Add door controllers, card readers, intercoms, air sensors to catalog

---

## V. Current Status

| Feature | Phase | Status |
|---|---|---|
| Canvas with draggable cameras | 1 | ✅ Done |
| FoV cones with ray-cast DORI zones | 1 | ✅ Done |
| Camera rotation + FoV angle drag | 1 | ✅ Done |
| Wall drawing + cone clipping | 1 | ✅ Done |
| Scale calibration tool | 1 | ✅ Done |
| Floor plan image upload | 1 | ✅ Done |
| Properties panel (resolution, tilt, retention…) | 1 | ✅ Done |
| Basic BOM table | 1 | ✅ Done |
| Bandwidth/storage calc | 1 | ✅ Done |
| Undo/redo, context menu, keyboard shortcuts | 1 | ✅ Done |
| Camera Model Picker modal (40+ models, 8 brands) | 2 | ✅ Done |
| Gradient FoV cone (single-color opacity layers) | 3 | ✅ Done |
| Wall types — Glass / Fence attenuation | 4 | ✅ Done |
| Focal length ↔ FoV binding | 5 | ✅ Done |
| Project total header + tabbed right panel | 6 | ✅ Done |
| BOM grouped by core + bulk edit + drag mode | 6 | ✅ Done |
| Per-camera storage/bandwidth details | 7 | ✅ Done |
| Site-wide storage summary + NVR recommendation | 7 | ✅ Done |
| Satellite map + address search | 8 | ✅ Done |
| Floor plan editor + opacity + map alignment | 8 | ✅ Done |
| 3D scene preview | 9 | ✅ Done |
| PDF report export (3-page) | 10 | ✅ Done |
| Save / load JSON plan | 11 | ✅ Done |
| URL share link (LZString) | 11 | ✅ Done |
| Core Type Picker + placement + properties | 12 | ✅ Done |
| Camera ↔ Core assignment (both directions) | 12 | ✅ Done |
| Add-On Picker (per-camera + per-plan scope) + BOM + properties | 13 | ✅ Done |
| Add-on ↔ Camera assignment (per-camera types, both directions) | 13 | ✅ Done |
| Toolbar restructuring (+ Device / + Camera) | 14 | ✅ Done |
| Left catalog panel removed | 14 | ✅ Done |
| CSV export | 6 | ✅ Done |
| PNG snapshot export | — | ✅ Done |
| Salesforce / Command sync | 15 | ⬜ |
| Access Control + Sensor devices | 15 | ⬜ |

---

## VI. Key Libraries (CDN links for prototype)

```html
<!-- Canvas -->
<script src="https://unpkg.com/konva@9/konva.min.js"></script>

<!-- Maps -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<!-- 3D -->
<script src="https://unpkg.com/three@0.160.0/build/three.min.js"></script>

<!-- PDF export -->
<script src="https://unpkg.com/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
<script src="https://html2canvas.hertzen.com/dist/html2canvas.min.js"></script>

<!-- URL sharing -->
<script src="https://unpkg.com/lz-string@1.5.0/libs/lz-string.min.js"></script>
```

---

## VII. Data Model

### State Object (key fields)
```javascript
state = {
  cameras: [],           // Array of camera objects
  cores: [],             // Array of core objects
  addons: [],            // Array of add-on objects
  userWalls: [],         // Array of wall segments
  selectedId: null,      // Selected camera ID
  selectedCoreId: null,  // Selected core ID
  selectedAddonId: null, // Selected add-on ID
  tool: 'select',        // Active tool: select | move | wall | place-camera | place-core | place-addon
  scalePPM: 10,          // Pixels per meter
  scaleCalibrated: false,
  nextId: 1,             // Auto-increment for cameras
  nextCoreId: 1,
  nextAddonId: 1,
  projectName: '...',
}
```

### Camera Object
```javascript
{ id, name, brand, model, formFactor, resolution, color, x, y, rotation,
  fov, detR, recR, idR, baseFocalLength, baseDetR, baseRecR, baseIdR,
  retention, license, payment, coreId, ndaa, eol, price }
```

### Core Object
```javascript
{ id, name, type, channels, color, x, y, payment, duration, price }
```

### Add-On Object
```javascript
{ id, typeId, name, type, color, price, cameraId,
  retention,  // per-camera storage types (30/60/90/180/365 days)
  payment,    // per-plan types: 'Upfront' | 'Yearly'
  duration    // per-plan Yearly types: 1/3/5/10 years
}
```

### ADDON_TYPES (8 types, 2 scopes)
```javascript
// Per-camera (scope:'camera') — assigned to specific cameras
'cloud-storage':  { price:500, fields:['camera','retention'], retentionOpts:[30,60,90,180,365] }
'external-nas':   { price:800, fields:['camera','retention'], retentionOpts:[30,60,90,180,365] }
'timelapse':      { price:200, fields:['camera','retention'], retentionOpts:[180,365] }
'cloud-metadata': { price:150, fields:['camera'] }
'cloud-snapshot': { price:100, fields:['camera'] }

// Per-plan (scope:'plan') — project-wide, not camera-specific
'alerts-storage':     { price:300, fields:['payment','duration'] }
'event-tags':         { price:250, fields:['payment','duration'] }
'video-verification': { price:400, fields:['payment','duration'] }
```

---

## VIII. Phase Size Guide

| Phase | Feature | Status |
|---|---|---|
| 1 | Canvas + FoV cones | ✅ Done |
| 2 | Camera Model Picker modal | ✅ Done |
| 3 | Gradient FoV cone visualization | ✅ Done |
| 4 | Wall types (Glass / Fence) | ✅ Done |
| 5 | Focal length ↔ FoV binding | ✅ Done |
| 6 | Enhanced BOM panel + CSV | ✅ Done |
| 7 | Storage/bandwidth calculator | ✅ Done |
| 8 | Satellite map + floor plan editor | ✅ Done |
| 9 | 3D scene preview | ✅ Done |
| 10 | PDF report export | ✅ Done |
| 11 | Save / Share | ✅ Done |
| 12 | Core devices | ✅ Done |
| 13 | Add-on devices (per-camera + per-plan) | ✅ Done |
| 14 | Toolbar restructuring | ✅ Done |
| 15 | Integrations (production) | ⬜ Not started |
