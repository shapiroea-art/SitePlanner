# Site Planner — Build Plan

## What We're Building (Simple Terms)

A digital site-planning tool — think Google Maps meets a drawing canvas. The user loads a floor plan or satellite map, places cameras, cores, and add-on devices onto it, and the tool automatically draws colored cones showing exactly what each camera can see (and what it misses). As devices are added, a live shopping list (BOM) builds itself on the side with prices, licensing, and storage estimates. The final output is a professional PDF report the user can send to a client.

It is equivalent to tools like IPVM's Camera Calculator or Genetec Site Designer, but built into Lumana.

> **Device types supported:** Video cameras, Lumana Cores (recording appliances), and Add-on devices (Cloud Storage, External Storage, Alerts Bucket). Access control devices (door controllers, readers) and sensors (air quality, intercoms) are planned for a future phase.

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
├── site-planner-prototype.html          ← Main prototype file (~5,600 lines, all phases built)
├── site-planner-location.html           ← Location management page
├── site-planner-sites.html              ← Sites list page
├── site-planner-plan.html               ← Plan list page
│
├── Knowledge/
│   └── SitePlanner-Plan.md              ← This file
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
- 3 types: Cloud Storage ($500), External Storage ($800), Alerts Bucket ($300)
- Capacity input, optional camera assignment (for Cloud Storage: camera search dropdown)
- Click to place on canvas, enter name, view properties

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
- **Camera → Core**: a camera can be assigned to one core; a core holds many cameras (up to its channel capacity).
- **Add-On → Camera**: an add-on has a `cameraId` field (one-to-one from add-on side). A camera can have multiple add-ons assigned (one-to-many from camera side). Found via `state.addons.filter(a => a.cameraId === camId)`.
- The camera properties panel shows both "Core Assignment" and "Add-on Assignment" blocks for managing these relationships.

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
    - Type dropdown: Cloud Storage ($500), External Storage ($800), Alerts Bucket ($300)
    - Capacity input field
    - Camera search dropdown (for Cloud Storage: filter/select cameras in project)
    - `ADDON_TYPES` constant, `ADDON_ICON` SVG constant

34. **[✅] Add-On Placement & Canvas Rendering**
    - Click to place on canvas → Konva group with colored rect body + storage icon (horizontal lines)
    - Inline name input after placement
    - Drag to reposition, click to select, selection ring highlight

35. **[✅] Add-On Properties Panel**
    - Name, type dropdown, color picker (8 colors), capacity input
    - Camera section (for cloud storage): shows assigned camera name (clickable → navigates to camera)
    - Camera assignment/removal functions
    - Delete button in panel header

36. **[✅] Add-On Assignment in Camera Properties**
    - "Add-on assignment" block after "Core Assignment" in camera properties
    - "Add add-on" button (aligned right) → modal with list of unassigned add-ons + "Create & assign" button
    - Assigned add-ons displayed as rows: icon, name/type/capacity, remove (×) button
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
| Add-On Picker + placement + properties | 13 | ✅ Done |
| Add-on ↔ Camera assignment (both directions) | 13 | ✅ Done |
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
{ id, name, type, capacity, color, x, y, cameraId, price }
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
| 13 | Add-on devices | ✅ Done |
| 14 | Toolbar restructuring | ✅ Done |
| 15 | Integrations (production) | ⬜ Not started |
