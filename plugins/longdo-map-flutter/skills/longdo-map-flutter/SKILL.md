---
name: longdo-map-flutter
description: >-
  Build mobile maps in Flutter (Android & iOS) with the official
  longdo_maps_api3_flutter SDK — adding the dependency, the LongdoMapWidget,
  handling the ready/click events via IJavascriptChannel, creating markers and
  overlays with Longdo.LongdoObject, and calling map methods through
  map.currentState. Use when writing or debugging Flutter code that uses Longdo
  Map. The SDK wraps the JS API3 in a WebView, so the longdo-map-js concepts
  apply too.
---

# Longdo Map — Flutter SDK (longdo_maps_api3_flutter)

Official getting-started: https://map.longdo.com/docs3/flutter/getting-start
Package: https://pub.dev/packages/longdo_maps_api3_flutter
Native API reference: https://api.longdo.com/map3/doc-native.html
Get a free API key: https://map.longdo.com/console/

> **Key mental model:** `longdo_maps_api3_flutter` is a thin wrapper that hosts
> the Longdo Map **JavaScript API v3** inside a `WebView`. You don't get Dart
> classes for every map feature — instead you call JS API methods by name and
> construct JS objects from Dart. So the patterns in the **longdo-map-js** skill
> (marker options, events, layers, overlays) are the source of truth for what's
> available; this skill is about the Dart ⇄ JS bridge.

## Add the dependency

```yaml
# pubspec.yaml
dependencies:
  longdo_maps_api3_flutter: ^1.12.0
```

The plugin pulls in `webview_flutter` and `package_info_plus`. Because it's
WebView-based, the app needs internet access at runtime:

- **Android:** ensure `<uses-permission android:name="android.permission.INTERNET"/>`
  is in `AndroidManifest.xml` (Flutter's default debug manifest has it).
- **iOS:** the WebView loads over HTTPS, so no extra ATS exceptions are needed
  for the default Longdo tiles.

> Verify the current minimum SDK / platform setup against the official
> getting-started page — native requirements can change between SDK versions.

## The map widget

`LongdoMapWidget` is driven through a `GlobalKey<LongdoMapState>` — you use the
key's `currentState` to call into the map after it's ready.

```dart
import 'package:flutter/material.dart';
import 'package:longdo_maps_api3_flutter/longdo_maps_api3_flutter.dart';

class MapScreen extends StatefulWidget {
  const MapScreen({super.key});
  @override
  State<MapScreen> createState() => _MapScreenState();
}

class _MapScreenState extends State<MapScreen> {
  final map = GlobalKey<LongdoMapState>();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: LongdoMapWidget(
          apiKey: "YOUR_API_KEY",   // inject from --dart-define / config, not hard-coded
          key: map,
          // map options forwarded to the JS longdo.Map constructor:
          options: const {
            "zoom": 14,
            "location": {"lon": 100.5018, "lat": 13.7563},
            "lastView": false,
          },
          // JS events you want surfaced back to Dart:
          eventName: [
            IJavascriptChannel(
              name: "ready",
              onMessageReceived: (message) => _onReady(),
            ),
            IJavascriptChannel(
              name: "overlayClick",
              onMessageReceived: (message) {
                // message carries the clicked overlay's data as a JSON string
                debugPrint("overlayClick: ${message}");
              },
            ),
          ],
        ),
      ),
    );
  }

  void _onReady() {
    // add a marker once the map's JS side has loaded
    final marker = Longdo.LongdoObject(
      "Marker",
      args: [
        {"lon": 100.5018, "lat": 13.7563},
        {"title": "Bangkok", "detail": "Hello from Flutter"},
      ],
    );
    map.currentState?.call("Overlays.add", args: [marker]);
  }
}
```

## Calling the map (Dart → JS bridge)

Everything routes through `map.currentState`:

| Dart call | What it does |
|-----------|--------------|
| `map.currentState?.call("Overlays.add", args: [marker])` | invoke a JS map method by dotted name |
| `map.currentState?.call("Layers.clear")` | call with no args |
| `map.currentState?.call("Route.search", args: [])` | trigger routing |
| `map.currentState?.objectCall(obj, "location")` | call a method **on a created object** (async) |
| `map.currentState?.run("/* raw JS */")` | execute an arbitrary JS string against the map |

Construct JS objects and statics from Dart:

```dart
// new longdo.Marker({lon, lat}, {options})
final marker = Longdo.LongdoObject("Marker", args: [
  {"lon": 100.5, "lat": 13.7},
  {"icon": {"html": "<div>…</div>", "offset": {"x": 0, "y": 0}}},
]);

// a JS static, e.g. longdo.Layers.LIGHT
final lightLayer = Longdo.LongdoStatic("Layers", "LIGHT");
map.currentState?.call("Layers.setBase", args: [lightLayer]);
```

The HTML-marker anchoring rule from **longdo-map-js** (outer div sized to the
real icon, no `position`, no negative margins, `offset:{x:0,y:0}` unless the
anchor isn't the icon's middle) applies verbatim — you're feeding the same JS
marker options, just from Dart maps.

## Gotchas

- **Wait for `ready`.** Calling `map.currentState?.call(...)` before the
  `ready` event fires runs against a half-loaded WebView and silently no-ops.
- **`currentState` can be null** while the widget is building — always
  null-check (`?.`).
- **Args are JSON.** Anything you pass in `args` must be JSON-serializable
  (maps, lists, primitives, or `LongdoObject`/`LongdoStatic` handles). You can't
  pass Dart closures into the map.
- **Event payloads arrive as strings.** `onMessageReceived` gives you the JS
  event serialized — parse it (`jsonDecode`) when you need fields.
- **Two older packages exist** (`longdo_maps_flutter`, `aliceblock/longdo_map`).
  Prefer `longdo_maps_api3_flutter` (published by longdo.com) for new projects.

## Reference

- Getting started: https://map.longdo.com/docs3/flutter/getting-start
- pub.dev package: https://pub.dev/packages/longdo_maps_api3_flutter
- Example app: https://pub.dev/packages/longdo_maps_api3_flutter/example
- Native API reference: https://api.longdo.com/map3/doc-native.html
- Underlying JS API concepts → **longdo-map-js** skill
- Blog walk-through: https://map-blog.longdo.com/longdomap-flutter/
