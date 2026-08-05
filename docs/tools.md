# Estimaite MCP Tool Reference

Complete reference for every tool the Estimaite MCP server exposes. Tools are thin wrappers over the same HTTP API the web app uses (no DB backdoor); each call runs **as the authenticated user**. Read `estimating_playbook` + `trade_estimating_guide` before touching any measurement tool — `download_takeoff_report` enforces the "complete is the only acceptable output" contract and will refuse a partial/sampled takeoff.

## Tool namespace — what the tool names look like

An agent never sees a bare `create_takeoff`. Tools arrive namespaced as `mcp__<server>__<tool>`:

| Deployment | Prefix | Example |
|---|---|---|
| Local stdio dev (`.mcp.json`) | `mcp__vision-local__` | `mcp__vision-local__create_takeoff` |
| Remote HTTP connector (`mcp.esti-maite.com`, "Estimaite") | `mcp__<connector-id>__` (e.g. `mcp__df03778f-…__`) | `mcp__df03778f-…__view_sheet` |

Throughout this doc tools are referred to by their bare name (e.g. `view_sheet`); prepend the right prefix for the server in play. **`upload_drawings` (local-disk PDF) and saving a report to `filePath` exist only on the local stdio server** (`allowLocalFs`); the remote HTTP server omits the disk tool and ignores `filePath` (returns the report inline instead), because the cloud host filesystem is not the user's machine.

Every measurement-flow tool (`create_estimate`, `view_sheet`, `create_takeoff`, `add_shape`, `add_shapes`, `auto_count`, `propose_segments`, `bind_tags`, `assemble_run`, `audit_sheet`, `list_run_ledger`, `trace_run`, `get_takeoff`, `get_estimate`, `update_estimate`) appends the non-negotiable **TAKEOFF RULES** guardrail text to its result.

---

## (a) Orientation & knowledge

Call these first, in this order, whenever asked to estimate/take off.

| Tool | Params | Returns / Use when |
|---|---|---|
| `whoami` | none | The current principal (org, user, role). Use to confirm identity/scope at session start. |
| `list_companies` | none | Companies in the current org. |
| `estimating_playbook` | none | The full professional takeoff procedure (contract, scale math, layer discipline, tracing, auto-count, QC). **READ FIRST** on any takeoff request. |
| `trade_estimating_guide` | `trade` (enum, **req**) | Trade-specific knowledge: system taxonomy, default material standards, what to count from schedules, accessory allowances, QC checklist. Call right after the playbook once the trade is known. Sheet prefix→trade: `M`=MECHANICAL, `P`=PLUMBING, `E`=ELECTRICAL, `FP`=FIRE_PROTECTION, `S`=STRUCTURAL, `A`-series→MASONRY/CARPENTRY/MILLWORK/PAINTING/FLOORING/ROOFING/MISC_METALS, `AD`/`D`=DEMOLITION, hazmat survey=ABATEMENT, `FO`=FUEL_OIL, `T`=TELECOM, `FA`=FIRE_ALARM, `SEC`/`SY`=SECURITY, `AV`=AUDIO_VISUAL. |

---

## (b) Project & drawings

### Projects

| Tool | Key params (**req**) | Returns / Use when |
|---|---|---|
| `list_projects` | none | All projects in the org. |
| `get_project` | `id`** | ONE project's full Details + `version`. Use the returned `version` as `expectedVersion` for `update_project`. |
| `create_project` | `name`**; `status?` (default ESTIMATING), `projectNumber?`, `squareFootage?`, `dueDate?` (ISO-8601), `customerId?` (link existing client), `clientName?`/`clientEmail?`/`assignedTo?` (quick free-text labels, no directory link), `description?` | New project. For linked client/assignee (chip + reporting rollup), create first then use `set_project_clients`/`set_project_assignees`. |
| `update_project` | `id`**; `expectedVersion?`, plus any Details field; pass `null` to CLEAR `dueDate`/`customerId`/`clientName`/`clientEmail`/`assignedTo` | Modify Details; omitted fields unchanged. |
| `delete_project` | `id`** | Deletes project + estimates/takeoffs. Irreversible — confirm with user first. |
| `list_bid_board` | none | Every project with estimate-VALUE rollup + clients + assignees (richer than `list_projects`). |

`status` enum: `ESTIMATING, BID_SUBMITTED, NO_BID, ACCEPTED, IN_PROGRESS, PAUSED, COMPLETE, DELAYED, LOST, ARCHIVED`.

### Org directory — clients (customers) & people (assignees)

| Tool | Key params (**req**) | Use when |
|---|---|---|
| `list_clients` | none | Find a client's id before linking. |
| `create_client` | `name`**, `email`**; `contactName?`, `phone?`, `address?` | Add client to org directory; returns id. |
| `set_project_clients` | `projectId`**, `customerIds`** (array; FIRST = primary; `[]` clears) | Replace-set a project's linked clients. |
| `list_people` | none | Find a person's id before assigning. |
| `create_person` | `name`**, `email`**; `color?` (hex chip, e.g. `#2563eb`) | Add assignable person; returns id. |
| `set_project_assignees` | `projectId`**, `personIds`** (array; `[]` clears) | Replace-set who is assigned. |

### Drawing upload (three paths)

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `upload_drawings` | `projectId`**, `filePath`** (accepts `~`) | **stdio/local only.** One rendered sheet per page, drawing numbers auto-detected. The normal way an estimate starts locally. |
| `request_drawing_upload` | `projectId`**, `fileName`** | Remote-safe step 1. Returns `{uploadUrl, uploadId}`. Then HTTP `PUT` the PDF bytes to `uploadUrl` with header `Content-Type: application/pdf`, then call `finalize_drawing_upload`. |
| `upload_drawing_base64` | `projectId`**, `fileName`**, `contentBase64`** | Small-PDF shortcut (**≤4 MB**; must start with `%PDF`). No PUT. Over-4 MB → use `request_drawing_upload`. |
| `finalize_drawing_upload` | `uploadId`** | Ingest after PUT. Waits briefly; small set → returns `{plans}`. Large set → `{status:"PROCESSING"}`; then poll `get_upload_status` until DONE (then `list_plans`) or FAILED. **Do NOT re-finalize the same uploadId** (409). |
| `get_upload_status` | `uploadId`** | Poll async ingest: PROCESSING / DONE / FAILED. |

### Plans (sheets)

| Tool | Key params (**req**) | Use when |
|---|---|---|
| `list_plans` | `projectId`** | List sheets. Each echoes its typical declaration (`typicalCount` / `typicalNote` / `representedByPlanId` — docs/18 L3). |
| `create_plan` | `projectId`**, `name`**; `drawingNumber?` | Placeholder sheet; returns plan id. |
| `reorder_plans` | `projectId`**, `orderedIds`** (must be EVERY current sheet id) | Reorder; dropdown groups by source PDF in this order. |
| `delete_plans` | `projectId`**, `ids`** (array) | Delete sheets + their takeoff marks (affected takeoffs recomputed). Confirm first — destroys work. |
| `set_plan_typical` | `planId`**, `typicalCount`**; `typicalNote?`, `representedByPlanId?` | Declare a sheet TYPICAL ×N (docs/18 L3): taken off ONCE, counts N times — the multiplier is ADDITIVE on shape-derived quantities (extraCount/drop modifiers count once; factor 1 reverts exactly), geometry is NEVER copied. Declarative: an omitted `typicalNote`/`representedByPlanId` CLEARS that field. `representedByPlanId` marks THIS sheet as represented (one level only — 400 on chains/cycles/self; it must stay geometry-free: a live shape on it is an audit punch item, dismissible kind `typical` only for a legitimate delta layer). `typicalCount>1` with no note is a punch item. Recomputes the sheet's takeoffs; the report renders `perUnit ×factor = total` + an auto-assumption line and requires a matching `typicalAttestation`. |

---

## (c) Scale — set BEFORE measuring anything

Quantities are meaningless until calibrated. Scale can be set per-**estimate** or per-**sheet** (sets often mix scales; enlarged details differ).

**Node space = the sheet rasterized at 150 DPI.** From a title-block scale: `k` (feet per node-pixel) `= 1 / (150 × paperInchesPerFoot)`. Examples: `1/8"=1'-0"` → `k=0.053333`; `1/4"=1'-0"` → `k=0.026667`; `3/32"=1'-0"` → `k=0.071111`.

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `calibrate_scale` | `estimateId`**; then EITHER `p1`,`p2`,`realLength` (2-point) OR `k` (ratio); `unit?`, `label?` | Estimate-wide scale. Recomputes all takeoffs. |
| `set_plan_scale` | `planId`**; EITHER `p1`,`p2`,`realLength` OR `k`; `unit?`, `label?` | ONE sheet's scale. Recomputes that sheet only. |
| `auto_detect_scale` | `estimateId`**, `planId`** | AI-read a printed scale note (server vision) → set estimate scale, recompute all. |
| `auto_detect_plan_scale` | `planId`** | AI-read the sheet's scale note → set + recompute that sheet. |

`p1`/`p2` are `{x,y}` in node space. Sanity-check against a known dimension (door ≈ 3 ft, parking stall ≈ 9 ft).

---

## (d) View the sheet — `view_sheet` · read its text — `get_sheet_text`

**Purpose:** return a PNG of a sheet (full or a cropped region) so the agent can read labels, confirm candidate segments (Set-of-Mark mode), and verify overlays. **This is the primary vision tool.**

| Param | Type | Req | Meaning |
|---|---|---|---|
| `planId` | string | ✔ | Sheet to view. |
| `x`,`y`,`w`,`h` | number | | Crop region in **node space**. Omit all → full sheet. |
| `width` | int 64–1568 | | Output image width px (default 1024). |
| `estimateId` | string | | **Overlay/verification mode:** renders that estimate's traced shapes in layer color over the sheet. Run an overlay pass after tracing. Required for `layer:'residual'`. |
| `takeoffId` | string | | Overlay ONLY that one takeoff's shapes — single-layer verification on a dense sheet (combinable with `estimateId`). |
| `layer` | `'full'`\|`'work'`\|`'work-clean'`\|`'residual'` | | `'work'` drops the screened architectural background, leaving only dark mechanical work AND returns `tags`. `'work-clean'` = work + tag-bubble/glyph stripping (the exact mask the geometry oracles read). `'residual'` (needs `estimateId`) = UNMEASURED work-layer ink painted red, each residue boxed — the visual form of `audit_sheet`'s residual check. `'full'` (default) = as-is. |
| `threshold` | int 40–245 | | Work-layer darkness cutoff (default 165). Lower if real material fades; raise if architecture bleeds through. |
| `marks` | boolean | | **Set-of-Mark mode** (use after `propose_segments`): paint the candidate segments ON the image, each a distinctly-colored polyline with an id chip (`s12`). You then SELECT ids for `assemble_run` instead of reading coordinates. ≤60 marks render (longest kept; the text block reports `markCount` + truncation) — narrow the filter or zoom rather than marking everything. |
| `ids` | string[] ≤60 | | Mark ONLY these segment ids — exactly the candidates you are deciding between. |
| `widthBand` | `"minPx-maxPx"` | | Mark only candidates whose MEDIAN width (node px) is in the band, e.g. `"14-30"` — match the size being traced. |
| `junctions` | boolean | | Also chip junction ids (`J4`) at the marked segments' endpoints. |
| `minLenPx` | number | | Mark only candidates at least this long (node px) — drops stubs/noise. |
| `grid` | boolean | | Labeled node-coordinate grid (magenta lines labeled with node x/y) — read node coords DIRECTLY off labels, no pixel math. For `auto_count` template boxes + the ASSUMED fallback only; the canonical linear path selects ids, not coordinates. |
| `gridStep` | int 5–2000 | | Grid spacing in node units (default auto ≈ 8 divisions). Smaller = finer placement. |

**Returned content:** an `image` item (base64 PNG) plus a `text` item with:

```json
{
  "planId": "…",
  "layer": "full" | "work" | "work-clean" | "residual",
  "sheetSize": { "width": <iw>, "height": <ih> },   // full sheet px at 150 DPI
  "region":    { "x": …, "y": …, "w": …, "h": … },   // node-space crop actually rendered
  "displayed": { "width": <ow>, "height": <oh> },    // output image px
  "toNodeCoords": "node.x = region.x + px * (region.w/displayed.width); node.y = region.y + py * (region.h/displayed.height)",
  "markCount": 18,                                   // marks mode: how many candidates rendered
  "marksTruncated": "…",                             // marks mode: present when the 60-mark cap dropped some
  "legend": [ "#00bcd4 = SA 24x12 — TDF wrapped", … ], // overlay mode: color → layer name
  "tags": [ { "str": "24x12", "x": <node>, "y": <node> }, … ]  // layer:'work'/'work-clean'
}
```

**Pixel→node mapping (needed only for `auto_count` templates + the ASSUMED `add_shape` fallback):**
```
scaleX = region.w / displayed.width
scaleY = region.h / displayed.height       // scaleX ≠ scaleY (outH is rounded server-side)
node.x = region.x + pixel_x * scaleX
node.y = region.y + pixel_y * scaleY
```
When `grid:true`, read coords straight off the magenta gridline labels — do not do pixel math. Never assume a single shared scale for x and y. Large/dense sheets that exceed ~3 MB base64 are auto-refetched at a smaller width. In `layer:'work'`/`'work-clean'`, `tags` gives duct/pipe sizes + CFM + notes with node positions so you segment runs by real size instead of guessing.

**Use when:** (1) survey the set (full sheet → quadrant crops); (2) `layer:'work'` first to comprehend systems + read sizes; (3) `marks:true` + `ids`/`widthBand` to confirm tag bindings and select run ids; (4) `estimateId` set for the mandatory overlay QC pass; (5) `layer:'residual'` to SEE what the audit says is unmeasured.

### `get_sheet_text` — positioned vector text as data

| Param | Type | Req | Meaning |
|---|---|---|---|
| `planId` | string | ✔ | Sheet to read. |
| `x`,`y`,`w`,`h` | number | | Clip region in node space — pass all four or none (whole sheet). |

Returns `items: [{str, x, y, h}]` — every size tag, CFM value, and note with its node position; `h` is the text height (small = tag/label, large = title). Zero OCR error — use it to read exact sizes and to sanity-check `bind_tags`. **409 on image-only sheets** (no source-PDF text) — read tags visually there.

---

## (e) Estimate & layers

### Estimates

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `list_estimates` | `projectId`** | List estimates. |
| `create_estimate` | `projectId`**, `name`** | New estimate (proposal); starts with a Default takeoff group. |
| `get_estimate` | `id`** | Estimate incl. pricing (`laborRate`, `markupPct`, `taxPct`) + proposal fields (`clientName`, `attention`, `scopeOfWork`, `inclusions`, `exclusions`, `notes`) + `projectId`. |
| `update_estimate` | `id`**; `laborRate?` ($/hr), `markupPct?` (PERCENT, 15=15%), `taxPct?` (PERCENT, on materials), `clientName?`, `attention?`, `scopeOfWork?`, `inclusions?` (string[]), `exclusions?` (string[]), `notes?`, `name?` (`null` clears) | Pricing + proposal settings. `laborRate` × each line's `laborPerUnit` = labor cost. |

### Takeoff groups (layer folders)

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `list_takeoff_groups` | `estimateId`** | List groups. |
| `create_takeoff_group` | `estimateId`**, `name`**; `parentId?` (nest), `order?` | e.g. "Ductwork — supply", "Hydronic piping". |
| `update_takeoff_group` | `groupId`**; `name?`, `parentId?` (`null`=root), `order?`, `multiplier?` | multiplier/parent change rescales the group's takeoffs. |
| `delete_takeoff_group` | `groupId`** | Takeoffs + child groups reparent up. |

### Takeoffs (layers) — the field set

`type` enum: `COUNT, LINEAR, LINEAR_DROP, LINEAR_AVG_DROP, COUNT_BY_DISTANCE, AREA, WALL_AREA, SEGMENT`.

**Every modifier a layer can carry** (shared by `create_takeoff`, `update_takeoff`, `create_takeoff_template`):

| Field | Type | Purpose |
|---|---|---|
| `color` | string | Hex layer color. **Give every layer a distinct color.** |
| `unit` | string | Display unit (`ft`, `ea`, `sf`…). |
| `unitPrice` | number ≥0 | $/unit (material). |
| `laborPerUnit` | number ≥0 | Labor hours/unit. |
| `wastePct` | number ≥0 | % waste on the quantity. |
| `profile` | MaterialProfile | Cross-section/material profile. |
| `trade` | Trade | Trade classification. |
| `category` | TakeoffCategory | Category classification. |
| `symbolId` | string | Symbol for COUNT stamps. |
| `lineWidth` | int | Canvas stroke weight. |
| `tag` | string | Free-text tag. |
| `multiplier` | number | Quantity multiplier. |
| `rise`,`run`,`slope`,`useHipAndValley` | number/bool | Slope geometry (roofs, sloped runs). |
| `dropLength`,`dropCount` | number/int | `LINEAR_DROP`: per-drop length + count. |
| `deviceCount` | int | `LINEAR_AVG_DROP`: devices for averaged drop. |
| `spacing` | number | `COUNT_BY_DISTANCE`: spacing between counted items. |
| `includeEndpoints` | bool | Count endpoints in distance-count. |
| `hasVolume`,`depth` | bool/number | `AREA`: give it volume (e.g. slab depth). |
| `wallHeight` | number | `WALL_AREA`: wall height. |
| `hasWeight`,`weightPerUnit` | bool/number | Weight rollup per unit. |

Type-specific modifier hints: `spacing`→COUNT_BY_DISTANCE; `dropLength`+`dropCount`→LINEAR_DROP; `dropLength`+`deviceCount`→LINEAR_AVG_DROP; `wallHeight`→WALL_AREA; `slope`+`rise`+`run`→sloped; `hasVolume`+`depth`→AREA.

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `list_takeoffs` | `estimateId`** | Each takeoff with `count` + `countPerPlan`. |
| `create_takeoff` | `estimateId`**, `name`**, `type`**; `groupId?` + any modifier above | Create a layer. |
| `update_takeoff` | `takeoffId`**; `name?`, `type?`, `groupId?`, `expectedVersion?` + any modifier | Change + recompute; pass only fields to change. |
| `get_takeoff` | `id`** | Full config + computed quantity. |
| `copy_takeoff` | `takeoffId`** | Duplicate the takeoff + all its shapes. |
| `delete_takeoff` | `takeoffId`** | Soft-delete takeoff + shapes. |
| `lock_takeoff` / `unlock_takeoff` | `takeoffId`** | Freeze/unfreeze editing. |
| `list_takeoff_templates` | none | Personal saved layer presets. |
| `create_takeoff_template` | `name`**, `type`** + any modifier | Save a reusable preset (incl. pricing). |
| `delete_takeoff_template` | `id`** | Remove a preset. |

#### ★ `create_takeoff` — the COMPLETE call (the #1 thing agents get wrong)

A takeoff line is **not bare geometry**. A size alone is not a line — the same 6" pipe prices differently as chilled water vs sanitary. Always set a **distinct color + real material name + dimensions/spec + unit + pricing**:

```json
// CONFIRMED linear layer — supply-air duct, fully specified
{
  "estimateId": "est_…",
  "groupId": "grp_ductwork_supply",
  "name": "SA 24x12 — TDF wrapped",
  "type": "LINEAR",
  "color": "#00bcd4",
  "unit": "ft",
  "profile": "RECTANGULAR_DUCT",
  "trade": "MECHANICAL",
  "category": "DUCTWORK",
  "unitPrice": 14.50,
  "laborPerUnit": 0.18,
  "wastePct": 10
}
```

```json
// COUNT layer — a scheduled device, priced per each
{
  "estimateId": "est_…",
  "groupId": "grp_equipment",
  "name": "VAV 5-13 — box w/ reheat",
  "type": "COUNT",
  "color": "#9c27b0",
  "unit": "ea",
  "symbolId": "vav-box",
  "trade": "MECHANICAL",
  "category": "EQUIPMENT",
  "unitPrice": 640,
  "laborPerUnit": 3.5
}
```

**ASSUMED-layer variant** — for a run with no label and no upstream size to inherit. Guess the size from drawn geometry, put it on its own red, double-weight layer so it screams on the canvas and can be deleted wholesale by the estimator:

```json
{
  "estimateId": "est_…",
  "name": "ASSUMED 18x10 — TDF wrapped",
  "type": "LINEAR",
  "color": "#ff0000",
  "lineWidth": 8,
  "unit": "ft",
  "profile": "RECTANGULAR_DUCT",
  "trade": "MECHANICAL",
  "category": "DUCTWORK"
}
```

Never mix assumed footage into a confirmed layer. One ASSUMED layer per guessed size+spec; list every one in the report's assumptions.

Distinct-color palette (never reuse adjacent hues): `#9c27b0 #00bcd4 #4caf50 #3f51b5 #ff9800 #e91e63 #795548 #607d8b`.

---

## (f) Assemble linear runs — `propose_segments` → `bind_tags` → `assemble_run` (THE canonical linear path)

Linear tracing is **select-not-draw** (docs/17): the server proposes candidate centerline segments (pure geometry, deliberately over-generated, zero semantics), you SELECT which ids form each run off marked views, the server welds + measures + stamps. **You never emit coordinates on a confirmed layer.** Work per SIZE-GROUP, never by page region.

### `propose_segments` — the geometry oracle

| Param | Type | Req | Meaning |
|---|---|---|---|
| `planId` | string | ✔ | Sheet to skeletonize. |
| `threshold` | number | | Work-mask darkness cutoff (default 165). **Propose at the DEFAULT before selecting** — `assemble_run` validates against the default-threshold graph; a custom threshold's ids will not chain. |
| `minLenPx` | number | | List only candidates at least this long (default 24; 0 = everything, noisy). |
| `ids` | string[] ≤200 | | Return FULL detail (incl. polyline) for just these ids. |

Returns `graphHash` (keep it — assembly requires it; stale hash → 409), `stats`, a summary list (≤400, longest first: `id, mode, lenPx, widthMed, strokeWeight, endA/endB {j, deg}, nearTags`), `bridges` (flagged gap-bridge candidates — never pre-welded), `junctionCount`. Every dark linear feature becomes a candidate — duct medial axes, pipe strokes, leaders, noise. Selecting is YOUR job; the proposer makes no judgment about what is a duct.

### `bind_tags` — read the tag to pick the duct

| Param | Type | Req | Meaning |
|---|---|---|---|
| `planId` | string | ✔ | Sheet. |
| `estimateId` | string | ✔ | The estimate whose scale converts tag inches → expected wall gap. |
| `threshold` | number | | Work-mask cutoff (default 165, same as `propose_segments`). |
| `radius` | number | | Candidate search radius around each tag in node px (default 150). |

For each parsed size tag: the width-matched candidates near it, ranked by distance (width matches on EITHER face ±35% — a 10x6 shows its 10in or 6in face in plan). Returns `graphHash`, `stats {tags, bound, none}`, `bindings`. **Confirm every binding you use by LOOKING** — `view_sheet marks:true ids:[candidates] layer:'work'` on a crop around the tag. **NONE is a legal answer**; a `none:true` tag is a punch item (the run may be missing from the graph → ASSUMED fallback).

### `assemble_run` — commit a selected run

| Param | Type | Req | Meaning |
|---|---|---|---|
| `takeoffId` | string | ✔ | The LINEAR layer for this size (one layer per system+size). |
| `planId` | string | ✔ | Sheet. |
| `graphHash` | string | ✔ | From `propose_segments`/`bind_tags`. Mismatch → 409 with the current hash: re-propose + re-select, never reuse old ids. |
| `segments` | string[] 1–200 | | The candidate ids forming ONE run — **order-free**, the server chains them. 400 on forks/loops/disconnected selections, **naming the offending ids** — drop the wrong one (usually a stroke shadow riding a band candidate, or a tap double-count) and retry. |
| `bridges` | `[id,id][]` ≤100 | | Gap bridges you EXPLICITLY confirmed. **Declare one for every consecutive pair not joined at a shared junction** — band (duct) and stroke (pipe) skeletons never share junctions, so real chains routinely need many (13 segments + 11 bridges on one 36.7 LF main). Over-declaring on a junction-joined pair is harmless (ignored). Undeclared gaps are never welded. |
| `boundaries` | `{start?, end?}` | | What each run END lands on — structured `{kind, note?}` with `kind` ∈ `terminal \| reducer \| equipment \| matchline \| open \| assumed` (note the far size on a reducer, e.g. `{kind:'reducer', note:'8x6'}`). Snapshotted with the stamped end coords as the run's durable endpoints (the circuit layer's input). A legacy bare string is recorded as `{kind:'open', note}`. |
| `replaceLedgerId` | string | | **RE-ASSEMBLY:** update this existing run-ledger row IN PLACE (same ledgerId — circuit edges keyed on it survive; endpoints re-snapshotted; old shape soft-deleted). Never delete+re-assemble by hand. Re-assembling after the run's circuit was reviewed makes that circuit STALE — re-run `propose_circuits` + `review_circuits` (the report gate blocks until you do). |
| `notes` | string | | Ledger notes (why these ids, what you verified). |
| `points` | `{x,y}[]` 2–50 | | **ESCAPE HATCH — layers named `ASSUMED…` only:** rough points snapped to the nearest graph endpoints when the run's segments are missing from the graph. Recorded as assumed (flagged, quarantine-only); refused on confirmed layers. |

Returns `lf` + `support` + `impliedSizeIn` + a ready-made verify view. **Cross-check before the next run:** low `support` = the weld sits over little ink — verify-view NOW, delete + re-select if off the duct. `impliedSizeIn` is **ADVISORY** — plan view shows ONE duct face (a 24x12 reads ~24 or ~12) — a mismatch means re-look, not auto-error.

### `list_run_ledger` — durable per-run state (resume from here)

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId`, `planId` | string | ✔ | The sheet's ledger. |

Each row records exactly which candidate ids + bridges were welded into which shape, with status + `graphHash`. **Call this FIRST on any resumed estimate.** A row whose `graphHash` ≠ `currentGraphHash` means the graph regenerated — its ids no longer name anything; re-run `audit_sheet` on that sheet before touching those runs.

### `audit_sheet` — the deterministic per-sheet gate

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId`, `planId` | string | ✔ | Sheet to audit. |
| `threshold` | number | | Work-mask cutoff (default 165). |

Pure geometry, no vision: (1) **residual ink** — dark work-layer linework claimed by no run (missed material, bboxed; SEE it with `view_sheet layer:'residual'`); (2) **unconsumed size tags** — a `12x8` tag with no 12x8 LINEAR geometry near it; (3) per-run **support** (hallucination check), **width vs declared size**, **mid-run width steps** (missed reducer), **dangling endpoints**, **contested ink** (two layers claiming the same linework); (4) **terminal orphans** (a counted device no run lands on); (5) **circuit checks** (only when circuits exist) — `circuits.attrMismatches` (a run's layer naming, or a colliding confirmed value reaching it through propagation, disagrees with its circuit's confirmed system) and `circuits.terminalConsistency` (a confirmed circuit attaching terminals from >1 layer family — possible cross-system joint); both gate-blocking until fixed (split/re-attribute) or dismissed with kind `circuit`. Returns the punch list + a numeric `gate` verdict; every item carries a stable `itemKey`. Run after each sheet's runs are assembled; FIX every item, quarantine genuine ambiguity to red ASSUMED, and re-run until `gate.pass` — **max 3 repair rounds**, then ASSUMED + findings. Requires a calibrated scale + source PDF (409 on image-only sheets). The report tool re-runs this gate and blocks on failures — see (k).

### `dismiss_audit_item` — journal a true non-scope/noise punch item

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId`, `planId` | string | ✔ | The sheet the item lives on. |
| `kind` | enum | ✔ | `residual \| faint \| tag \| run-width \| dangling \| terminal-orphan \| contested \| circuit` |
| `itemKey` | string | ✔ | Exactly as `audit_sheet` returned it (circuit keys: `circ:<circuitDbId>:attr:<ledgerId>` / `circ:<circuitDbId>:terminals`). |
| `reason` | string ≥8 | ✔ | Specific + checkable — printed in the report. |
| `undo` | boolean | | Revive a wrongly-dismissed item. |

Dismissals are journaled, survive resume, drop the item from the gate counts, and print in the audit + the final report. Dismiss ONLY what is truly not scope or truly not work — never unfinished work; dismiss circuit items only after READING the disagreeing statements (e.g. a transfer-duct circuit that legitimately touches two families).

---

## (f2) Circuit pass — `propose_circuits` → look (`marks:'circuits'`) → `review_circuits` (docs/18 L1)

After a sheet's runs assemble + audit green: cluster runs + counted terminals into CIRCUITS (connected networks), review every proposed joint, and attribute the SYSTEM per circuit — one sourced verdict propagates to every member run. Code proposes and floods; **you rule on every joint and every attribute**. The report's SYSTEMS section (per confirmed circuit: system + source, size chain, LF by size, terminals by family, open ends, missed reducers) is built from this pass, and the report gate blocks on unreviewed or stale circuits.

### `propose_circuits` — deterministic over-proposed clustering

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId` | string | ✔ | The estimate whose assembled runs + counted terminals to cluster. |
| `planId` | string | | Limit the pass to one sheet (default: every sheet with assembled runs). |

Returns `circuitSetHash` (keep it — `review_circuits` 409s on a stale one), `circuits` (`id` 'c3', `status`, `color`, member `runs` with layer + lf, `terminals`, `lfTotal`, `openEnds`, `attributes.system.proposed` = RANKED EVIDENCE — never auto-confirmed), `edges` (`id` 'e12', `kind` `coincident|sizeStep|terminal`, `distPx`, `status`), `unattachedTerminals` (the completeness oracle — a counted point with no circuit means its feed was never traced), and `propagation` (per-run assignments, conflicts, size chains, `missedReducers`). Re-proposing is safe: confirmed/rejected verdicts survive; a confirmed edge whose geometry moved re-opens (`edgesRegrown`); a circuit whose member runs changed AFTER review comes back `stale:true` (the report gate blocks on it). **Open ends are a DISCOVERY WORK LIST** — each is a missed connection (confirm the joint), a missed run (trace it), or a genuine boundary (re-assemble with `replaceLedgerId` + the right endpoint `kind`).

### `review_circuits` — ordered confirm / split / attribute verdicts

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId` | string | ✔ | |
| `circuitSetHash` | string | ✔ | From `propose_circuits` — proves you reviewed the live geometry (stale → 409: re-propose). |
| `verdicts` | array 1–100 | ✔ | Each: `{circuitId, verdict, …}` — `circuitId` = 'c3' or db id; `verdict` ∈ `confirm \| split \| attribute`. |
| — `confirm` | `edgeIds?: string[]` | | Batch-ratifies the circuit's ELIGIBLE proposed edges (each ratified edge prints with its `distPx`) and sets the circuit confirmed. Vetoed edges return under `needsIndividual` with the reason — **WIDE** joint (distPx > joinRadius/2), **TEXT-NEAR** (sheet text ≤30px of the joint — read it first), **DIFFERING TERMINAL EVIDENCE** (possible cross-system joint, e.g. 10x6 SA beside 10x6 EA). View each, then confirm by name (`edgeIds:['e12']`) or reject via split. Never blind-ratify a vetoed joint. |
| — `split` | `rejectEdgeIds: string[]` | ✔(split) | Rejects the named edges and re-clusters the circuit into fragments. A rejected edge NEVER re-bridges (survives re-propose) — one wrong supply-to-exhaust joint is undone with a single verdict. There is no merge tool: a missed merge stays a flagged open end. |
| — `attribute` | `name, value, source{kind, ref?}` | ✔(attr) | Record what you READ off the sheet — free strings (e.g. `name:'system', value:'SA'`) + the source (`kind: tag\|legend\|schedule\|note`, `ref`: what/where). Stored beside the machine-proposed evidence (weigh it, never rubber-stamp it). Confirmed attributes PROPAGATE across confirmed edges to every member run; the response shows assignments, conflicts (block the audit gate until resolved), size chains, missedReducers. |

Verdicts process in order; each failure reports in `outcomes` and the rest continue. Resume-safe: verdicts persist immediately.

### Seeing circuits — `view_sheet marks:'circuits'`

`view_sheet { planId, estimateId, marks:'circuits' }` paints every assembled run in its circuit's color: `c3` chips at circuits, `▸e12` chips at PROPOSED joints, solid dots at confirmed, rejected hidden. Narrow with `circuitIds:['c3']` on dense sheets. ALWAYS look before ruling.

---

## (g) Draw shapes — `add_shape` / `add_shapes` (fallback + counts/areas; ASSUMED-only for linear)

For LINEAR geometry this is the **documented fallback**, not the canonical path — use it only when the graph cannot serve (image-only sheet, or a run genuinely missing from the graph), and ONLY onto a red `ASSUMED` layer. COUNT points and AREA boundaries still use it normally. Shapes are drawn in **node space** (= full-render 150-DPI pixels). Convert displayed `view_sheet` pixels back with `toNodeCoords` first (or read off the `grid:true` labels).

**Nodes format:** `nodes: { outer: [{x,y}, …], holes?: [[…]], control?: [{x,y}] }` — `outer` is the ordered point list (min 1, max 20000 points; `holes`/`control` optional, same ceilings).

Geometry per takeoff type:
- **LINEAR** (duct/pipe run — **ASSUMED-layer fallback only**; confirmed layers go through `assemble_run`): `outer` = ordered centerline vertices, a node at every elbow/tee/transition. Start a branch's first point ON the main's centerline so the network connects. Stop at a reducer and continue on the new size's layer from that exact point.
- **COUNT** (device): `outer` = a single point `[{x,y}]` at the device symbol (use `shape:'POINT'`).
- **AREA / WALL_AREA**: `outer` = the closed boundary ring (vertices in order); `holes` = openings to subtract.

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `add_shape` | `takeoffId`**, `planId`**, `nodes`**; `shapeId?`, `shape?` (`GENERIC\|RECTANGLE\|CIRCLE\|ARC\|FREEFORM\|POINT`), `label?` | Add/update ONE shape. Re-sending the same `shapeId` updates it. Returns recomputed count. |
| `add_shapes` | `takeoffId`**, `shapes`** (array 1–1000 of `{planId, nodes, shape?, shapeId?, label?}`) | **STRONGLY PREFERRED** for more than a couple shapes: one round-trip + ONE recompute. Returns takeoff + created/updated counts. |
| `delete_shape` | `takeoffId`**, `shapeId`** | Delete a shape + recompute. Use to remove hallucinated geometry. |
| `list_shapes` | `takeoffId`** | Raw drawn geometry — verify exactly what was measured. |

```json
// LINEAR run via add_shape — ASSUMED fallback layer only (confirmed layers: assemble_run)
{ "takeoffId": "tko_…", "planId": "pln_…",
  "nodes": { "outer": [ {"x":812,"y":640}, {"x":812,"y":905}, {"x":1140,"y":905} ] } }
```
```json
// Many COUNT stamps in one call
{ "takeoffId": "tko_…",
  "shapes": [
    { "planId": "pln_…", "shape": "POINT", "nodes": { "outer": [ {"x":410,"y":298} ] } },
    { "planId": "pln_…", "shape": "POINT", "nodes": { "outer": [ {"x":560,"y":301} ] } }
  ] }
```

**Rule 3 (non-negotiable):** every drawn point must sit on a real element. Immediately re-view each region in overlay (`view_sheet` + `estimateId`) and `delete_shape` anything floating over nothing.

---

## (h) Count — `auto_count`

**Purpose:** template-match one clean symbol instance and stamp every match on a COUNT takeoff. **Output is a DRAFT you must overlay-verify** — it template-matches CAD linework and mis-places on dense sheets.

| Param | Type | Req | Meaning |
|---|---|---|---|
| `takeoffId` | string | ✔ | The COUNT takeoff to stamp. |
| `planId` | string | ✔ | Sheet to scan. |
| `template` | object | ✔ | `{x,y,w,h}` in node space — a **tight** box around ONE clean instance (`w,h` ≥4). |
| `template.ignore` | object | | Node-space sub-rect INSIDE the template to mask a part that VARIES per instance (a circled tag digit, a changing CFM value) so the match keys on the stable outline + fixed label. Without it, off-template instances score low and the count comes up short. |
| `threshold` | 0.3–0.99 | | Min match correlation (default 0.72). Dense sheets over-match — use a tight box and **0.85+**. |
| `detail` | int 8–48 | | Fine-match resolution (default ~28, already accurate). Rarely needed; raise toward 40+ for very fine/dense symbols if a count looks low, lower (~16) to trade accuracy for speed. |

Returns match count + updated quantity. **Always** then `view_sheet` with `estimateId` to overlay-verify stamped points: delete false positives, manually stamp misses. If the number is wildly high, the template over-matched — clear points and rerun with a tighter box / higher threshold.

---

## (i) Query & QC

| Tool | Key params (**req**) | Returns |
|---|---|---|
| `get_takeoff_quantity` | `takeoffId`** | `{name, type, unit, unitKind, count, countPerPlan, volume, weight, perimeter}` — the computed quantity + per-sheet breakdown. |
| `get_takeoff` | `id`** | Full layer config + quantity. |
| `list_shapes` | `takeoffId`** | Raw geometry drawn on the layer. |
| `list_takeoffs` | `estimateId`** | All layers with `count` + `countPerPlan`. |
| `get_estimate` | `id`** | Pricing + proposal settings + `projectId`. |

QC checks: `countPerPlan` per layer (a 10× anomaly usually = scale mistake); every schedule type has a reconciled COUNT layer; every layer's unit is right (`ft` linear, `ea` counts); group multipliers intended. The mandatory final step is the fresh-eyes pass: `audit_sheet` green on every traced sheet (or failures quarantined/dismissed with reasons) plus an overlay pass over **every** sheet looking for dark runs with no line (false negatives) and lines on nothing (false positives).

---

## (j) Pricing — cost catalog

The org's shared material + labor unit-cost catalog used on the Estimating screen.

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `list_cost_items` | none | The shared pricing catalog. |
| `create_cost_item` | `name`**; `category?`, `trade?`, `unit?`, `unitPrice?` ($/unit material), `laborPerUnit?` (hrs/unit), `wastePct?`, `notes?` | Add a catalog item. |
| `update_cost_item` | `id`**; same fields (`null` clears) | Update a catalog item. |
| `delete_cost_item` | `id`** | Remove a catalog item. |

Per-line pricing lives on the takeoff itself (`unitPrice`/`laborPerUnit`/`wastePct`); estimate-wide pricing (`laborRate`, `markupPct`, `taxPct`) is set via `update_estimate`.

---

## (j2) Spec engine — `list_specs` → `assign_spec` → `explain_takeoff_pricing` (docs/18 L2)

**Assign specs BEFORE pricing dimensioned mechanical lines.** The ENGINE owns the trade math — gauge by size, seal class by pressure, connection standard, lbs/LF, the split waste + fitting-weight allowances — as versioned, auditable rules. Never hand-derive numbers these rules own: an engine-shaped line (HVAC trade + dimension/weight columns) with no assignment prints an **un-audited-pricing warning finding** in the report.

### `list_specs` — the rule catalog

| Param | Type | Req | Meaning |
|---|---|---|---|
| `trade` | string | | Filter by trade key, e.g. `'mechanical'`. |

Returns `{engineVersion, rules:[{key, name, trade, mode, version, inputSchema, outputSchema, scope, note}], tables:[{key, name, trade, verified, version, source}]}`. `mode` ∈ `formula` (deterministic standard) | `heuristic` (allowance — always flagged) | `auditBand` (sanity check, never prices) | `replacedByL1` (allowance that DISARMS once counted fittings exist in scope). `verified:false` on a table forces an assumption line on every use. Call this before `assign_spec` to pick the chain.

### `assign_spec` — dry-run calculator + pricing write-through

| Param | Type | Req | Meaning |
|---|---|---|---|
| `projectId`, `takeoffId` | string | ✔ | ONE takeoff per assignment. |
| `specKeys` | string[] (1–50) | ✔ | Ordered rule keys, e.g. `['mech.duct.gauge','mech.duct.sealClass','mech.duct.connection','mech.duct.insulation','mech.duct.lbsPerLF','mech.duct.wastePct','mech.duct.fittingWeightPct','mech.duct.totalLbs']` — earlier outputs feed later inputs. |
| `inputs` | record | | Agent-confirmed SEMANTIC inputs only, e.g. `{system:'SA', pressureClass:2, exposure:'indoor'}`. Geometry (width/height/diameter/LF) auto-fills from the takeoff columns — do not re-type it. A 400 lists exactly which inputs to confirm (with allowed enum values). |
| `overrides` | record `{name:{value, source, note?}}` | | Output override with a REAL cited source (spec section / addendum). Journaled + forces an assumption line. Never use one to silence a number. |
| `apply` | `{unitPrice?, laborPerUnit?, wastePct?}` booleans | | Which pricing columns the engine writes. **OMIT for a pure dry-run** (nothing persisted — the agent's calculator). |
| `priceBasisCostItemId` | string | | CostItem to price from: unit `'lb'` → `unitPrice = $/lb × bare lbs/LF` and the allowance % lives in `wastePct` once (dollars = totalLbs × $/lb, no double-count); other units pass through. |
| `force` | boolean | | Override the clobber guard AFTER reviewing the divergence with the user. |
| `remove` | boolean | | Delete the assignment (pricing columns untouched — the line prices as hand-set again). |

Returns `{dryRun, applied, applyNotes, outputs, assumptions, disarmed, firedRuleKeys, resolvedInputs, autoFilled, resultHash, ruleVersions, …}`. The fitting-weight allowance auto-**disarms** when L1 counted fittings exist (counted elbows replace the bump — do not re-add it). **Clobber guard:** hand-edited pricing columns since the last apply → the call returns `divergence` (hand value vs engine value) and writes NOTHING — review with the user, then drop that field from `apply` or re-call with `force:true`.

### `explain_takeoff_pricing` — the trust artifact (survives a bid review)

| Param | Type | Req | Meaning |
|---|---|---|---|
| `projectId`, `takeoffId` | string | ✔ | The assigned takeoff. |

Returns the assignment snapshot: rule chain + pinned `ruleVersions`, confirmed inputs vs auto-filled geometry, outputs, the **byte-stable evaluation trace exactly as priced**, every assumption (defaults, unverified tables, heuristics, overrides), the CostItem basis math, `fieldProvenance` per pricing column (engine-applied vs hand-edited-after-apply vs hand-set), and a live `stale` check (geometry moved since apply → the applied price is dead; re-run `assign_spec` with the same specKeys/inputs). 404 = no assignment (the takeoff is hand-priced). Run it after every apply and before the report.

**How it reaches the report:** every assumption prints in the report's **"Engine assumptions (specs)"** ledger (deduped, with fire counts + a superseded-rule-versions line); spec-stale findings merge into `audit_sheet` and the report gate's findings.

---

## (k) Report — `download_takeoff_report`

**Purpose:** produce the finished takeoff report (the deliverable). Enforces the completeness contract — **it refuses incomplete/sampled takeoffs.**

| Param | Type | Req | Meaning |
|---|---|---|---|
| `estimateId` | string | ✔ | The estimate to report. |
| `format` | `md`\|`csv`\|`json` | | Default `md` (human), `csv` (Excel), `json` (structured). |
| `filePath` | string | | Save to the user's machine (e.g. `~/Downloads/estimate.md`). **stdio only** — over HTTP it's ignored and the report returns inline. |
| `assumptions` | string (md bullets) | | Every default/assumption applied (`md` only). |
| `findings` | string (md bullets) | | Discrepancies (e.g. schedule vs plan counts) (`md` only). |
| `exclusions` | string (md bullets) | | **ONLY** genuine scope exclusions or truly illegible drawings — **NEVER** density/effort skips. |
| `coverage` | array `{sheet, status, note?}` | (req for a complete report) | ONE entry per plan sheet; `status` ∈ `traced\|counted\|traced-and-counted\|schedule-only\|legend-only\|not-in-scope`. EVERY sheet must appear. |
| `acknowledgeIncomplete` | boolean | | Set true ONLY for a legitimately scope-limited takeoff (user asked for one small system). Requires `incompleteReason`. |
| `incompleteReason` | string | | Real scope boundary — NOT density/effort. |
| `acceptAuditFailures` | boolean | | The **numeric audit gate** (gate v2) re-runs `audit_sheet` on every sheet carrying LINEAR/SEGMENT geometry and blocks the report while any fails. Set true ONLY after genuinely attempting the fixes; requires `auditFailureReason`. Accepted failures print in the report as findings — they become the customer-visible error bound. Never use it to hide unfinished work. |
| `auditFailureReason` | string | | Why the remaining audit failures are acceptable (e.g. "residual is architecture bleed-through on a heavily screened sheet, visually confirmed") — printed as a finding. |
| `typicalAttestation` | array of strings | (req when any sheet is typical) | ONE entry per sheet declared `typicalCount>1` (`set_plan_typical`), naming the sheet AND the exact factor and standing behind the identity claim — e.g. `"M-113 ×7: L2–L8 identical per plan note 3"`. Cross-checked against the DB declarations: a missing, factor-mismatched, or phantom entry (naming a non-typical sheet) blocks the report. |
| `unattributedAttestation` | string | | The honesty gate (docs/18 WP-H4) blocks when >10% of the linear footage reaches NO confirmed system. Set ONLY when the share is genuinely irreducible (the documents never name the system) — the reason prints in the report next to the number. Attribute (`review_circuits`) or quarantine to ASSUMED first; density/effort are never valid. |

**What it refuses (returns `{blocked:true, …}`):**
1. **Shortcut prose** — any of `assumptions/findings/exclusions/incompleteReason/coverage[].note` containing a density/effort tell (`representative`, `sampled`, `a subset`, `main systems only`, `too dense`, `for brevity`, `ran out of time`, `did not trace all`, etc.).
2. **Structural triviality** (unless `acknowledgeIncomplete`): no takeoffs; **no COUNT layers** (equipment never counted); fewer than 3 total shapes.
3. **Coverage gaps:** missing sheets not attested; or a sheet attested `traced/counted` that has **no actual takeoff geometry** (cross-checked against `countPerPlan` — you cannot claim a sheet you never measured).
4. **Audit-gate failures** (numbers, not prose): any traced sheet whose `audit_sheet` gate fails (residual ink, unconsumed tags, low support, dangles, contested ink, open circuit punch items…) blocks the report unless `acceptAuditFailures:true` + `auditFailureReason` — in which case the failures are merged into `findings`. Sheets the audit cannot run on (image-only, no scale) become advisory notes, never silent skips.
5. **Circuit gate** (only when the estimate has circuits — pre-L1 estimates unaffected): (a) **STALE circuits** — the API recomputes every circuit's hash from the LIVE run endpoints at report time; a mismatch means a run was re-assembled after the circuits were reviewed → blocked until `propose_circuits` + `review_circuits` re-run (verdicts on unchanged joints survive). (b) **UNREVIEWED circuits** — a circuit blocks while its status is still `proposed` OR any of its joints is still `proposed` (a re-propose after moved geometry re-opens joints inside a confirmed circuit — they must be re-ruled), unless every member run is quarantined on an `ASSUMED` layer.

6. **Spec gate — split posture (docs/18 WP-H4 ENFORCING):** BLOCKING: any **fitting scope-overlap** — an assignment whose `replacedByL1` weight allowance FIRED while counted FittingRecords now exist in its scope (the trap-patch-1 double count). Checked ESTIMATE-WIDE (works even when no sheet is auditable); not dismissible — re-run `assign_spec` on the named takeoff so the engine disarms the allowance. ADVISORY (warning by spec): (a) mechanical (HVAC) takeoffs with dimension/weight columns but **no live spec assignment** append an un-audited-pricing warning finding; (b) **spec-stale** assignments (geometry/inputs moved after apply) append per-line findings.

7. **Typical gate** (only when a sheet declares `typicalCount>1` — non-typical estimates unaffected): every DB typical declaration must be attested by name + exact factor in `typicalAttestation`; missing, factor-mismatched, or phantom entries block. The per-sheet `typical` audit punch items (live shape on a represented sheet, missing note, deleted representer) block through the normal audit gate (4).

8. **Honesty gate** (docs/18 WP-H4, ENFORCING — only when the estimate has circuits or spec assignments): (a) **ledger parity** — Σ printed assumption-line fire counts must equal the raw firing total; a mismatch means an engine firing was dropped from the printed ledger and blocks with no escape; (b) **unattributed-LF share ≤10% or attested** — footage reaching no confirmed system blocks until attributed, quarantined to ASSUMED, or attested via `unattributedAttestation` (which prints as a finding); (c) the gate **completes the error bound**: it converts the sheet audits' open residual ink to an LF-equivalent and prints `bound = width-slack + residual LF-equiv × avg confirmed $/LF` in the header line (unauditable sheets are counted in a finding, never silently omitted).

On success it merges your `assumptions/findings/exclusions` into the document, and — when circuits exist — a **SYSTEMS section** (per confirmed circuit: system + source, size chain, LF by size, terminals by family, open ends, missed reducers; unconfirmed circuits print as one RED pending line), and — when fitting data exists — a **FITTINGS section** with mode provenance per line (`counted(L1)` from FittingRecords vs `allowance(rule:<key>)` from fired spec rules, the circuit each count belongs to; scope overlaps echo RED), and — when spec assignments exist — an **ENGINE ASSUMPTIONS (SPECS) ledger** (every default / unverified-table / heuristic / override the engine fired, deduped by rule + text with fire counts and inputs, plus a superseded-rule/table-versions line when a pinned standard moved), and — when a sheet is typical — a **TYPICAL SHEETS section** (per-sheet quantities render `perUnit ×factor = total`, one auto-assumption line per declaration), and — when circuits or spec assignments exist — the **error-bound header line** `CONFIRMED $X (±bound) · ASSUMED $Y (flagged) · UNATTRIBUTED $Z (RED)` plus an **ATTRIBUTE HONESTY section** (the DoD ratio, its printed definitions, LF buckets, and the documented bound formula). To pass: finish every in-scope sheet, count against every schedule, work every sheet's audit punch list to `gate.pass`, finish the circuit pass (f2), assign specs on engine-shaped lines (j2), declare typical sheets (`set_plan_typical`) and attest them, run the whole-sheet overlay QC, then supply a `coverage` entry for every sheet. **The finish line is a green gate + the printed bound — never prose accuracy claims.**

---

## (l) Admin & analytics

| Tool | Key params (**req**) | Notes |
|---|---|---|
| `lock_takeoff` / `unlock_takeoff` | `takeoffId`** | Freeze/unfreeze a finished layer. |
| `copy_takeoff` | `takeoffId`** | Duplicate takeoff + shapes. |
| `delete_takeoff` | `takeoffId`** | Soft-delete. |
| `delete_takeoff_group` | `groupId`** | Delete folder (children reparent up). |
| `delete_takeoff_template` | `id`** | Remove saved preset. |
| `delete_plans` | `projectId`**, `ids`** | Delete sheets + marks. Confirm first. |
| `delete_project` | `id`** | Delete project + all children. Irreversible — confirm first. |
| `get_analytics` | `from?`, `to?` (ISO dates; omit → trailing 12 mo) | Bid-pipeline rollup: KPIs (pipeline value, win rate, $/sqft), status/funnel breakdowns, time-series, by-client/estimator/trade, deadline buckets. |
| `list_bid_board` | none | Every project with estimate-value rollup + clients + assignees. |

---

## Legacy fallback — `trace_run` (never the canonical path)

The tag-anchored wall-follower that predates the propose/select/audit architecture. **Demoted:** use it only as an optional quick check of a single run on sheets where the segment graph fails (and `assemble_run` points/`add_shape` ASSUMED fallbacks don't fit) — never as the primary tracing method, and always overlay-verify its output. Canonical linear work goes through section (f).

| Param | Type | Req | Meaning |
|---|---|---|---|
| `takeoffId`, `planId` | string | ✔ | The LINEAR takeoff + sheet. |
| `sizeTag` | string ≤40 | (preferred) | Duct size as tagged, e.g. `"10x6"`, `"24x12"`, `"8 Dia"`. |
| `seed` | `{x,y}` | (preferred) | ONE node-space point on the tagged duct. Server confirms a duct of that size is there, then auto-traces both directions until it reduces, hits a terminal, or ends. |
| `reducerTol` | 0.05–0.9 | | Gap drift before it counts as a size change (default 0.22). |
| `terminalFactor` | 1.2–6 | | Gap > this × run median ⇒ terminal/device (default 2.2). |
| `path` | `{x,y}[]` 2–2000 | (legacy) | Rough centerline (vertex per bend) to snap one segment. Prefer `sizeTag`+`seed`. |
| `mode` | `auto`\|`vector`\|`raster` | | Linework to snap onto (default `auto`). |
| `threshold` | 40–245 | | Raster work-layer darkness cutoff (default 165). |
| `maxHalfWidth`,`corridor`,`gap`,`extend` | number | | Raster/vector wall-finding + gap-bridging knobs. |

Returns the clean centerline, updated quantity, `stopReasonStart`/`stopReasonEnd` (`reducer|terminal|edge`), and `impliedSizeIn` (verify it matches `sizeTag`). Scale must be set first for tag-anchored mode; needs a source-PDF sheet. Always overlay-verify with `view_sheet` + `estimateId`.

---

## Canonical end-to-end order

1. `whoami` → `estimating_playbook` → `trade_estimating_guide` (resuming? `list_run_ledger` first per worked sheet — graphHash mismatch ⇒ re-audit before editing)
2. `create_project` (or pick) → upload (`upload_drawings` / `request`+`finalize` / `base64`) → `list_plans`
3. `create_estimate` → set scale (`calibrate_scale` / `auto_detect_scale`; per-sheet if mixed)
4. `view_sheet` (`layer:'work'`) + `get_sheet_text` to study set + schedules
5. **Counts first:** COUNT layers per schedule → `auto_count` → overlay-verify
6. Classify → `create_takeoff` (distinct color + material + dims + unit + pricing) per system/size, grouped
7. **Assemble (select-not-draw), per sheet:** `propose_segments` (default threshold, keep `graphHash`) → `bind_tags` → confirm bindings on marked views → per SIZE-GROUP select ids + declare bridges → `assemble_run` (cross-check `lf`/`support`/`impliedSizeIn`)
8. **Audit each sheet:** `audit_sheet` → work the punch list (`layer:'residual'` to see misses; `dismiss_audit_item` for typed non-scope dismissals; red ASSUMED for ambiguity) until `gate.pass` (max 3 rounds)
9. **Circuit pass:** `propose_circuits` → `view_sheet marks:'circuits'` → `review_circuits` (confirm/split every joint + a SOURCED system attribute per circuit) → work the open-end list + circuit punch items; re-review after any re-assembly (stale circuits block the report)
10. **Spec pass (j2):** `list_specs` → `assign_spec` per dimensioned mechanical line (confirm semantic inputs; dry-run, then `apply` + `priceBasisCostItemId`; overrides only with a cited source) → `explain_takeoff_pricing` (must show in-sync)
11. **Typical sheets (docs/18 L3):** when the set says "LEVELS 2–8 IDENTICAL", take the representative sheet through its audit gate, then `set_plan_typical {planId, typicalCount: N, typicalNote: "<cite the plan note>"}`; uploaded represented levels get `{typicalCount: 1, representedByPlanId}` and stay geometry-free — never fake ×N with group multipliers or copied layers
12. Fresh-eyes QC: `get_takeoff_quantity`, `list_takeoffs`, whole-sheet overlay pass (zero false pos/neg)
13. `update_estimate` pricing/proposal → `download_takeoff_report` (md + csv) with full `coverage` (audit gate + circuit gate must pass, or `acceptAuditFailures` + reason → printed findings; unassigned engine-shaped lines + spec-stale print as warning findings; typical declarations need `typicalAttestation` entries naming sheet + exact factor)

**Source of truth:** `/Users/nikolas/Documents/Vision/apps/mcp/src/index.ts` (tool registrations + zod schemas), `/Users/nikolas/Documents/Vision/apps/mcp/src/playbook.ts` (workflow + contract), `/Users/nikolas/Documents/Vision/packages/api-client/src/index.ts` (return shapes).
