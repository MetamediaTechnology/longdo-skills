---
name: longdo-map-js
description: >-
  Tips, patterns, and gotchas for building web maps with the Longdo Map v3
  JavaScript API — initialisation, HTML markers that don't drift on zoom,
  directional markers, clustering, click handling, popups, UI components, layer
  control, geolocation, and place tags. Use when writing or debugging
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

## Loading the API and initialising a map

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

## HTML marker icons that don't drift on zoom

**The problem:** `offset: { x: -W/2, y: -H/2 }` (negative values to centre a
large icon) causes visible zoom-drift in Longdo Map v3. The larger the icon,
the worse it looks.

**The fix:** always use `offset: { x: 0, y: 0 }` and shift the icon via CSS
margins instead.

```javascript
const SIZE = 48; // icon pixel size
const html =
  // Outer div: NO position property — this is critical for drift-free placement.
  // Negative margins centre the box around the geo pin (offset 0,0 = top-left at pin).
  `<div style="width:${SIZE}px;height:${SIZE}px;margin-left:-${SIZE / 2}px;margin-top:-${SIZE / 2}px">` +
    // Inner div: position:relative — safe popup/ring anchor, does NOT affect Longdo's placement.
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
- The **Longdo icon root** (outermost HTML element) must have **no `position`
  property**. Even `position:relative` with `offset:{0,0}` can reintroduce drift.
- Centre via `margin-left` / `margin-top` (negative), not via the `offset` param.
- Any inner `position:relative` div is fine — it only affects children, not
  Longdo's placement.

---

## Directional beak (heading arrow) that stays upright

Rotate a beak polygon inside the SVG around the icon's centre point. The vehicle
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
    `<div style="width:48px;height:48px;margin-left:-24px;margin-top:-24px">` +
      `<div style="position:relative;width:48px;height:48px;cursor:pointer;` +
                  `filter:drop-shadow(0 2px 5px rgba(0,0,0,.5))">` +
        `<svg width="48" height="48" viewBox="0 0 48 48">` +
          beak +
          `<circle cx="${cx}" cy="${cy}" r="${r}" fill="${color}" stroke="rgba(255,255,255,0.92)" stroke-width="2"/>` +
          // glyph (a 15×15 SVG path centred in the puck) goes here, NO rotation applied
        `</svg>` +
      `</div>` +
    `</div>`
  );
}
```

---

## Cluster / bubble markers

Clusters don't need precise centering, so `offset:{x:0,y:0}` works as-is
(top-left at geo pin).

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

**Do NOT call `marker.location({ lat, lon })` on HTML markers with a centred
icon.** `location()` repositions the marker without reapplying the icon offset,
causing drift.

Instead, remove and recreate the marker when its position changes:

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
// CSS (popup floats above the icon, centred horizontally)
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

## Speed-state colour palette

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
