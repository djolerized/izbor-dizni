PROJECT: WordPress plugin "Izbor Rasprskivača" (Nozzle Selector)

GOAL
Build a WordPress plugin that lets users choose a crop (kultura) and see growth stages (faze rasta). For each stage, show recommended nozzles (dizne) above the stage timeline/image. A nozzle can be used across multiple crops and multiple stages and multiple treatments, and can be placed multiple times (repeatable).

STACK / RULES
- WordPress plugin (PHP 8+), no theme dependency.
- Use CPT + taxonomies + post meta (ACF optional but plugin must work without ACF).
- Use admin pages inside wp-admin.
- Use nonce, capability checks, sanitization/escaping.
- Use AJAX for drag&drop saving in admin.

DATA MODEL

1) Custom Post Type: ir_crop (Kultura)
- Labels: "Kulture"
- Supports: title, thumbnail, editor (optional)
Meta:
- ir_crop_order (int) optional ordering
- ir_crop_image_id (int) optional main image if not using featured
- ir_crop_slug (string) optional

2) Custom Post Type: ir_stage (Faza rasta)
- Labels: "Faze rasta"
- Supports: title
Meta:
- ir_stage_crop_id (int) REQUIRED (parent crop post ID)
- ir_stage_order (int) REQUIRED (order on timeline)
- ir_stage_label_short (string) optional (e.g. "2 lista")
- ir_stage_desc (textarea) optional

3) Custom Post Type: ir_nozzle (Dizna)
- Labels: "Dizne"
- Supports: title, thumbnail
Meta (all optional unless stated):
- ir_nozzle_image_id (int) if not using featured
- ir_nozzle_jet_image_id (int) image of spray pattern
- ir_nozzle_application_time (string) e.g. "od 2 lista do cvetanja" (DISPLAY ONLY)
- ir_nozzle_material (string) e.g. "DELRIN"
- ir_nozzle_pressure_min (float) e.g. 2
- ir_nozzle_pressure_max (float) e.g. 5
- ir_nozzle_speed_min (float) e.g. 8
- ir_nozzle_speed_max (float) e.g. 12
- ir_nozzle_flow_min (int) e.g. 140
- ir_nozzle_flow_max (int) e.g. 200

4) Taxonomy: ir_treatment (Tretman)
- Attached to: ir_application (see below)
- Terms: Herbicidi, Fungicidi, Insekticidi... (admin can manage)

5) Custom Post Type: ir_application (Primena dizne)
This is the key many-to-many link object.
- Supports: title (auto-generated)
Meta (REQUIRED):
- ir_app_crop_id (int) crop post ID
- ir_app_stage_id (int) stage post ID (belongs to that crop)
- ir_app_nozzle_id (int) nozzle post ID
Meta (optional):
- ir_app_time_text (string) e.g. "na crno; od 2 lista do cvetanja"
- ir_app_note (textarea)
- ir_app_order (int) ordering within a stage (for display)
Taxonomy:
- ir_treatment assigned to this application (one or multiple terms allowed)

IMPORTANT: Repeatable nozzle placement
- Do NOT enforce uniqueness on (crop_id, stage_id, nozzle_id, treatment). Allow duplicates.
- Each placement is its own ir_application post, so the same nozzle can be placed multiple times.

ADMIN UI REQUIREMENTS

A) Admin Menu
Add top-level menu "Izbor rasprskivača" with submenus:
- Kulture (list/manage ir_crop)
- Faze rasta (list/manage ir_stage)
- Dizne (list/manage ir_nozzle)
- Primene (list/manage ir_application)
- "Raspored dizni" (custom builder page)

B) Custom Builder Page: "Raspored dizni"
Purpose: Visual drag&drop assignment of nozzles to a crop, ABOVE the crop image/timeline, NOT over it.

UI Layout:
1. Dropdown: Select Crop (kultura)
2. When crop selected:
   - Show Crop image (if exists)
   - Above the image: a dedicated "placement area" (workspace) with fixed height (e.g. 180px) where nozzles will be placed.
   - Below/overlapping conceptually: a horizontal stage timeline bar showing the crop stages in order.
   - For each stage, define a vertical column/segment in the workspace where nozzle markers can be dropped.

3. Left side panel (or top panel): searchable list of all nozzles with thumbnail + name.
   - Drag a nozzle from list into a stage segment in the workspace.
   - The dropped item becomes a "marker card" showing nozzle name + small image.
   - Each marker has controls:
     - Treatment dropdown (taxonomy terms)
     - Time text input (ir_app_time_text)
     - Remove button
     - Duplicate button (creates another application with same settings)

4. Styling/CSS critical fix:
- Ensure marker wrapper has fixed width; e.g. .izbor-dizni-marker { width: 50px; } (or another constant).
- Prevent markers stretching 100% width.

C) Saving Logic
- On each drop / edit / reorder, save via AJAX:
  Endpoint: wp_ajax_ir_save_layout
  Payload includes:
    crop_id
    list of placements [{application_id (0 if new), stage_id, nozzle_id, treatment_term_ids[], time_text, order, x(optional), y(optional}]
- Create or update ir_application posts accordingly.
- If application removed in UI -> delete that ir_application post (or mark as trash).
- Reordering inside a stage updates ir_app_order.

D) Validation
- stage must belong to selected crop.
- capability: manage_options for admin builder.
- nonce required.
- sanitize all inputs.
- escape output.

FRONTEND REQUIREMENTS

Shortcode:
[izbor_rasprskivaca]
Attributes:
- show_details="modal|inline" default modal
- default_crop_id=""

Frontend flow:
1) Show crop selection (cards or dropdown).
2) When crop chosen:
   - Load stages (ordered by ir_stage_order).
   - For each stage:
     - Query ir_application where ir_app_crop_id = crop_id AND ir_app_stage_id = stage_id
     - Order by ir_app_order ASC
     - Render nozzle markers above stage.
3) Clicking a nozzle marker opens details:
   - nozzle title, nozzle image, jet image
   - material, pressure range, speed range, flow range
   - show application context: treatment terms + time_text (from the application record)

Frontend Data Loading:
- Use REST endpoint or AJAX:
  wp_ajax_nopriv_ir_get_crop_data and wp_ajax_ir_get_crop_data
Return JSON:
- crop {id,title,image_url}
- stages [{id,title,order}]
- placements grouped by stage: [{application_id, stage_id, nozzle_id, nozzle fields, treatment names, time_text, order}]

PERFORMANCE
- Cache crop JSON in transient for 5 minutes; invalidate when crop/stage/nozzle/application updated.

SECURITY
- Nonce checks for all AJAX.
- Capability checks for admin actions.
- Escape all frontend HTML and attributes.

PLUGIN STRUCTURE
/izbor-rasprskivaca/
  izbor-rasprskivaca.php (main plugin file)
  /includes/
    cpt.php
    taxonomies.php
    meta-fields.php
    admin-menu.php
    admin-builder-page.php
    ajax-admin.php
    ajax-frontend.php
    shortcode.php
    helpers.php
  /assets/
    /css/admin.css
    /css/frontend.css
    /js/admin-builder.js
    /js/frontend.js

ACCEPTANCE TESTS
1) Create 2 crops, each with 5 stages.
2) Create 5 nozzles.
3) In builder: add same nozzle 3 times to crop A (same stage and different stages). Ensure allowed.
4) Add same nozzle to crop B. Ensure independent.
5) Set treatment + time_text per placement; verify saved and shown on frontend.
6) Verify markers do not stretch full width; wrapper width fixed (e.g. 50px).
7) Frontend shortcode shows correct crop selection, correct stages, and markers above each stage.
8) Clicking marker shows nozzle details + application context.

DELIVERABLES
- Zip-ready WordPress plugin folder.
- Works without external dependencies (ACF optional).
- Clean uninstall: remove CPT registrations only (do not delete posts by default).

END.
