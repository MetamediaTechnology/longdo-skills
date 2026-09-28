---
name: longdo-map-js
description: >-
  Tips, patterns, and gotchas for building web maps with the Longdo Map v3
  JavaScript API — initialization, HTML markers that sit exactly on their
  coordinate (fixing apparent zoom-drift), directional markers, clustering,
  click handling, popups, UI components, layer control, geolocation, and place
  tags. Use when writing or debugging
  browser/web code that uses the Longdo Map JavaScript API. For the location
  REST APIs (search, geocode, routing, traffic) see the longdo-map-rest skill.
---

# Longdo Map v3 JavaScript API — Tips & Tricks

Official docs: https://map.longdo.com/docs/javascript/
Get a free API key: https://map.longdo.com/console/

These are battle-tested patterns and non-obvious gotchas. Prefer them over
naive approaches — several work around real quirks in the v3 renderer.

> The Longdo Map v3 JavaScript API is built on top of **MapLibre GL JS**. The
> underlying MapLibre map instance is exposed as `map.Renderer`, so you can call
> any MapLibre method directly on it (e.g. `map.Renderer.getStyle()`,
> `map.Renderer.setLayoutProperty(...)`, `map.Renderer.on(...)`) when the Longdo
> wrapper doesn't cover what you need.

> Need server-side search, geocoding, routing, or traffic data? Those are REST
> calls — see the **longdo-map-rest** skill. This skill is the browser map only.

---

## Loading the API and initializing a map

Include the script with your key, then create the map after the page and the
API have loaded. Bind to the `ready` event before touching the map.

```html
<!-- Never hard-code a real key in committed source. Inject it server-side
     or from an environment variable / config that is not in version control. -->
<script src="https://api.longdo.com/map3/?key=YOUR_API_KEY"></script>

<div id="map" style="width:100%;height:100%"></div>
<script>
  let map;
  window.onload = () => {
    map = new longdo.Map({
      placeholder: document.getElementById('map'),
      layer: [longdo.Layers.LIGHT],        // or longdo.Layers.NIGHT
      ui: longdo.UiComponent.MobileWithFullLayerSelector,
      lastView: true,                      // restore the user's last position
    });
    map.Event.bind('ready', init);
  };

  function init() {
    map.zoom(14);
    map.zoomRange({ min: 5, max: 20 });
    map.location({ lon: 100.5018, lat: 13.7563 }, true);
  }
</script>
```

> Security: the key is exposed to the browser by design, but keep it out of your
> repo. Load it from an env var / server-side template and restrict the key to
> your domain(s) in the Longdo console.

---

## HTML marker icons that sit exactly on their coordinate

**The problem:** a large HTML icon appears to sit on the right spot when you are
zoomed in, then looks badly misplaced when you zoom out — as if it drifts with
zoom. It does not drift. It has a **constant pixel offset**, and a constant
24 px error is a few meters of ground at z19 but over a kilometer at z9, which
is what makes it look zoom-dependent.

**The cause:** the outermost element of your `icon.html` — usually a `position`
on it, sometimes a size that doesn't match the drawn icon or a stray margin.

An `icon.html` marker is handed straight to `maplibregl.Marker` with your markup
inside its element, and MapLibre centers that element on the coordinate with
`translate(-50%, -50%)`. `position: absolute` (or `fixed`) takes your icon out of
flow, the wrapper collapses to 0×0, the `-50%` centers nothing, and the icon's
**top-left** lands on the pin instead of its middle.

**The fix:** give the outer element an explicit `width` / `height` equal to the
icon's real size, no `position`, no margins, and `offset: { x: 0, y: 0 }`. The
map then centers it exactly; `offset` is only for icons whose anchor is not the
middle (see the table below).

```javascript
const SIZE = 48; // icon pixel size
const html =
  // Outer div: sized to the real icon, NO position, NO margin — the map
  // centers this box on the coordinate for you.
  `<div style="width:${SIZE}px;height:${SIZE}px">` +
    // Inner div: position:relative — safe popup/ring anchor, affects only its
    // own children, not the map's placement of the outer box.
    `<div style="position:relative;width:${SIZE}px;height:${SIZE}px;cursor:pointer">` +
      `<svg width="${SIZE}" height="${SIZE}" viewBox="0 0 ${SIZE} ${SIZE}">` +
        `<circle cx="24" cy="24" r="12" fill="#27B24B" stroke="white" stroke-width="2"/>` +
      `</svg>` +
    `</div>` +
  `</div>`;

const marker = new longdo.Marker(
  { lat: 13.7563, lon: 100.5018 },
  { icon: { html, offset: { x: 0, y: 0 } }, title: 'Vehicle', detail: 'Speed: 60 km/h' }
);
map.Overlays.add(marker);
```

**Rules:**
- Give the **Longdo icon root** (outermost element of `icon.html`) an explicit
  `width` / `height` equal to the icon's **real visible size**. MapLibre wraps
  it in its own div and computes the centering from that size, so a root that is
  smaller or larger than what you draw (an unsized div, a 48 px box holding a
  64 px SVG) is centered on the wrong box.
- The root must have **no `position`**. `position:relative` is harmless;
  `absolute` / `fixed` collapse the wrapper and put the icon's top-left on the
  pin.
- Do **not** add negative margins to center it. On a correctly built icon they
  overshoot: a 48 px icon with `margin:-24px` lands 12 px up and 12 px left
  (the margin shrinks the wrapper the map already centered, so you get half of
  it back).
- Use `offset` only when the point that must touch the ground is **not** the
  icon's middle:

  | Icon | Anchor point | `offset` |
  |------|--------------|----------|
  | home, circle, badge, cluster bubble | middle | `{ x: 0, y: 0 }` |
  | pin / teardrop, tip at the bottom edge | bottom-center | `{ x: 0, y: -H/2 }` (48 px → `y: -24`) |

- Any inner `position:relative` div is fine — it only affects children.

**If you really need `position:absolute` on the root**, treat it as a separate
approach, not a tweak of the one above: the wrapper collapses to 0×0, so the
pin is at the icon's top-left, and you center it yourself with
`margin-left:-W/2; margin-top:-H/2` (still `offset: {0, 0}`, or shift the
margins for a non-center anchor). Pick one approach per icon — never mix a
sized, unpositioned root with centering margins, or `absolute` with the map's
own centering.

**Debugging a misplaced icon:** movement of sub-pixel to ~1 px while zooming is
projection and pixel rounding, and is expected. Anything larger is not drift —
it is a constant offset from the markup. Check, in order: the root's
**size** (matches the drawn icon?), **margin** (any?), **position** (any on the
root?), and **offset** (non-zero only for a non-center anchor?).

> Measured on `api.longdo.com/map3/` at both renderer versions (`?v=old` →
> MapLibre 2.4.0 and the current default → 5.7.1), icon sizes 48 px and 200 px,
> static at nine zooms and across 114 frames of animated zoom with pitch and
> bearing. Every configuration held its error constant to within 0.35 px, which
> is MapLibre's pixel rounding — there is no zoom-drift to design around. The
> only thing that varies between configurations is the constant offset:
> no `position` → 0 px, negative margins → 12 px, `position:absolute` → 24 px.

---

## Directional beak (heading arrow) that stays upright

Rotate a beak polygon inside the SVG around the icon's center point. The vehicle
glyph stays upright because it is NOT inside the rotated `<g>`.

```javascript
function vehicleIconHtml(heading, color) {
  const cx = 24, cy = 24, r = 12;
  // heading: 0 = north, 90 = east, 180 = south, 270 = west
  const beak = heading >= 0 && heading < 360
    ? `<g transform="rotate(${heading},${cx},${cy})">
         <polygon points="${cx},2 ${cx - 6},13 ${cx + 6},13"
                  fill="${color}" stroke="rgba(255,255,255,0.75)" stroke-width="0.8"/>
       </g>`
    : '';
  return (
    `<div style="width:48px;height:48px">` +
      `<div style="position:relative;width:48px;height:48px;cursor:pointer;` +
                  `filter:drop-shadow(0 2px 5px rgba(0,0,0,.5))">` +
        `<svg width="48" height="48" viewBox="0 0 48 48">` +
          beak +
          `<circle cx="${cx}" cy="${cy}" r="${r}" fill="${color}" stroke="rgba(255,255,255,0.92)" stroke-width="2"/>` +
          // glyph (a 15×15 SVG path centered in the puck) goes here, NO rotation applied
        `</svg>` +
      `</div>` +
    `</div>`
  );
}
```

---

## Cluster / bubble markers

Same rule as any HTML icon — the outer element sized to the bubble (the
`box-shadow` rings don't count; they don't affect layout), no `position`, and
`offset:{x:0,y:0}` centers the bubble on the cluster's coordinate.

```javascript
function clusterHtml(count) {
  const sz = count > 5000 ? 72 : count > 500 ? 56 : 44;
  const color = count > 5000 ? '#D8392B' : count > 500 ? '#F5821F' : '#2F80ED';
  return `<div style="width:${sz}px;height:${sz}px;border-radius:50%;` +
         `background:${color};color:#fff;font-weight:700;font-size:${sz * 0.28}px;` +
         `display:flex;align-items:center;justify-content:center;` +
         `box-shadow:0 0 0 6px ${color}26,0 0 0 12px ${color}12,0 4px 12px rgba(0,0,0,.4)">` +
         `${count >= 1000 ? (count / 1000).toFixed(1) + 'k' : count}</div>`;
}

map.Overlays.add(new longdo.Marker(
  { lat, lon },
  { icon: { html: clusterHtml(1234), offset: { x: 0, y: 0 } }, clickable: false }
));
```

---

## Marker click handling

Use `map.Event.bind('overlayClick', ...)` instead of DOM click listeners.
Store a custom id on the marker object to identify which one was clicked — custom
properties survive map operations.

```javascript
const m = new longdo.Marker({ lat, lon }, { icon: { html, offset: { x: 0, y: 0 } } });
m._myId = 'vehicle-001';  // custom property — survives map operations
map.Overlays.add(m);

map.Event.bind('overlayClick', ov => {
  const id = ov?._myId;
  if (!id) return;
  console.log('clicked:', id);
});
```

---

## Marker position update

`marker.location({ lat, lon })` moves an HTML marker in place and keeps its
placement exact — measured to within a pixel over repeated moves and across
zooms, on an icon built to the rule above. Prefer it: it is far cheaper than
rebuilding the element, and it keeps any popup, drag state and DOM listeners.

```javascript
marker.location({ lat, lon });
```

If the icon's **appearance** must change too (a new heading, a new color), the
markup is what changes, so rebuild it:

```javascript
map.Overlays.remove(oldMarker);
const newMarker = new longdo.Marker({ lat, lon }, { icon: { html, offset: { x: 0, y: 0 } } });
newMarker._myId = oldMarker._myId;
map.Overlays.add(newMarker);
```

---

## Popup anchored to a marker

Append the popup to the **inner** `position:relative` div (not the Longdo root):

```javascript
// CSS (popup floats above the icon, centered horizontally)
// .my-popup { position:absolute; left:50%; bottom:calc(100% + 4px); transform:translateX(-50%); ... }

function openPopup(innerDivId, html) {
  const wrap = document.getElementById(innerDivId);
  if (!wrap) return;
  const el = document.createElement('div');
  el.className = 'my-popup';
  el.innerHTML = html;
  el.addEventListener('click', e => e.stopPropagation());
  wrap.appendChild(el);
}
```

Longdo also has a built-in `longdo.Popup` overlay if you don't need a custom
DOM-anchored popup.

---

## Speed-state color palette

Consistent with Thai traffic conventions used on Longdo Map itself:

```javascript
function speedColor(kmh) {
  if (!kmh || kmh < 5) return '#E5392E'; // Stop — red
  if (kmh < 20)        return '#F5821F'; // Slow — orange
  if (kmh < 40)        return '#F2C61E'; // Med  — amber
  return                      '#27B24B'; // Flow — green
}
```

---

## UI components

```javascript
// Pick a preset at construction time:
//   longdo.UiComponent.Full
//   longdo.UiComponent.Mobile
//   longdo.UiComponent.MobileWithFullLayerSelector  // forces the layer selector
//                                                    // (road, admin) on mobile
const map = new longdo.Map({ placeholder, ui: longdo.UiComponent.MobileWithFullLayerSelector });

// Toggle individual widgets at runtime:
map.Ui.Crosshair.visible(true);
map.Ui.Toolbar.visible(true);
map.Ui.Zoombar.visible(true);
map.Ui.DPad.visible(true);
map.Ui.Geolocation.visible(true);
```

**Trigger geolocation programmatically** (simulate a click on the locate button):

```javascript
map.Ui.Geolocation._geolocateButton.click();
```

---

## Hide vector layers for a clean display

The base map is a vector style — you can hide individual layers (buildings,
road labels, admin labels, etc.) instead of switching the whole basemap.

```javascript
// Inspect what's available first:
console.log(map.Renderer.getStyle().layers);

// Then hide what you don't want:
map.Renderer.setLayoutProperty('building', 'visibility', 'none');
map.Renderer.setLayoutProperty('label_route', 'visibility', 'none');
map.Renderer.setLayoutProperty('label_admin', 'visibility', 'none');
```

For a quick clean look without per-layer tweaking, use the light basemap:
`longdo.Layers.LIGHT`.

---

## Place tags (POI filtering)

Longdo can display points of interest filtered by tag (e.g. `bank`,
`restaurant`, `hotel`, `pharmacy`, plus Thai-language tags like `สถานศึกษา`
(education) and `ท่องเที่ยว` (tourism) — 40+ categories).

See the tag reference and full list: https://map.longdo.com/docs/javascript/marker/tag

---

## Custom Map Layer Menu (longdo.LayerSelector)

In enterprise application development, relying on the map platform's default UI components often limits flexibility. The native Longdo Map `LayerSelector` automatically bundles all Longdo Map and third-party layers into a single pre-defined layout.
To achieve complete data governance and tailored user experiences, developers need the ability to build a Custom Layer Selector. Implementing a custom selector allows you to override the default component completely, enabling you to:
- Enforce Whitelisting: Explicitly define which layers are accessible (e.g., hiding paid/premium "Non-free" layers like Google Maps or GISTDA to eliminate accidental API billing risks).
- Tailor UI/UX: Rearrange, rename, or group layers (such as Vector vs. Raster) to match your application's design system and user workflows.
- Control Layer Logic: Dynamically manage how baseline map data and informational overlays (like real-time traffic) interact with one another.

Using the official `longdo.MenuBar` class, we can cleanly disable the native picker and inject a custom-tailored layer orchestration interface.

```javascript
// 1. Initialize the Longdo Map instance
var map = new longdo.Map({
  placeholder: document.getElementById("map"),
});

// 2. Configure the Custom UI once the map is fully loaded
map.Event.bind("ready", function () {
  
  // Cleanly hide the default system Layer Selector
  map.Ui.LayerSelector.visible(false);
  
  // Construct a brand new MenuBar with permitted layers only
  const customLayerSelector = new longdo.MenuBar({
    button: [
      { label: 'แผนที่', value: 'NORMAL', type: longdo.ButtonType.Radio },
      { label: 'ดาวเทียม', value: 'HARD', type: longdo.ButtonType.Radio },
      { label: 'จราจร', value: 'TRAFFIC', type: longdo.ButtonType.Radio }
    ],
    dropdown: [
      { label: 'พาสเทล', value: 'PASTEL', type: longdo.ButtonType.Radio },
      { label: 'พาสเทล ถนนเทา', value: 'PASTEL_GRAY', type: longdo.ButtonType.Radio },
      { label: 'สว่าง', value: 'LIGHT', type: longdo.ButtonType.Radio },
      { label: 'กลางคืน', value: 'NIGHT', type: longdo.ButtonType.Radio },
      { label: 'มืด', value: 'DARK', type: longdo.ButtonType.Radio },
      { label: 'เขตการปกครอง', value: 'POLITICAL', type: longdo.ButtonType.Radio },
      { label: 'OpenStreetMap (Longdo)', value: 'OSM', type: longdo.ButtonType.Radio },
      { label: 'Raster', type: longdo.ButtonType.Group },
      { label: 'สถานที่', value: 'RASTER_POI', type: longdo.ButtonType.Radio },
      { label: 'ภาพถ่าย', value: 'THAICHOTE', type: longdo.ButtonType.Radio },
      { label: 'OpenStreetMap', value: 'RASTER_OSM', type: longdo.ButtonType.Radio }
    ],
    dropdownLabel: "อื่นๆ",
    change: (currentMenuItem, lastMenuItem) => {
      // Cleanup Logic: Remove the overlay layer if switching away from TRAFFIC
      if (lastMenuItem && lastMenuItem.value === 'TRAFFIC') {
        map.Layers.remove(longdo.Layers[lastMenuItem.value]);
      }

      // Layer Switching Logic
      switch (currentMenuItem.value) {
        case 'TRAFFIC': {
          // Traffic overlay requires a solid base layer underneath (e.g., LIGHT)
          map.Layers.setBase(longdo.Layers.LIGHT);
          map.Layers.add(longdo.Layers[currentMenuItem.value]);
          break;
        }
        default: {
          // Standard base layer switch
          map.Layers.setBase(longdo.Layers[currentMenuItem.value]);
          break;
        }
      }
    }
  });
  
  // 3. Inject the custom component into the map UI container
  map.Ui.add(customLayerSelector);
});
```

When customizing your MenuBar, you can freely mix, match, and map your own text labels to any underlying layer key supported by the platform. For a complete, up-to-date reference of all pre-defined layer strings (such as NORMAL, TRAFFIC, WMS, etc.) provided by the API, please refer to the official documentation: [Longdo Map API Documentation - Layers Section](https://api.longdo.com/map3/doc.html#Layers)

---

## Stretching low-zoom layer imagery (API2)

Display a raster layer at deeper zoom levels than it natively provides by
stretching lower-resolution tiles:

```javascript
var bluemarble = new longdo.Layer('bluemarble_terrain', { source: { min: 1, max: 11 } });
```

---

## Reference

- JavaScript API docs: https://map.longdo.com/docs/javascript/
- Live examples: https://mapdemo.longdo.com/
- Official tips blog: https://map-blog.longdo.com/longdo-map-api-tips/
- API console (get a key): https://map.longdo.com/console/
- Server-side search / geocode / routing / traffic → **longdo-map-rest** skill
- Quick example: https://mapdemo.longdo.com/index-simple.php
