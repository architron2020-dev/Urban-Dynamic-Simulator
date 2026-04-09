# Urban 3D Simulator — claude.md

> Project documentation for AI-assisted urban simulation and analysis.
> This file describes the architecture, data flows, analysis logic, and prompting conventions for the Urban 3D Simulator built with Three.js and the Open-Meteo API.

---

## Project overview

The Urban 3D Simulator is a browser-based urban analysis tool that renders a procedural 3D city, applies real-time climate data from any location on Earth, and overlays environmental analysis layers (traffic, heat island, air quality, isovist, daylight, space syntax, noise, accessibility). It is designed to help urban designers, planners, and researchers rapidly prototype and compare urban interventions without requiring GIS software or fixed simulation functions.

The core design principle is **dynamic unpredictability**: access conditions, environmental stress, and urban performance metrics shift continuously based on live weather, time of day, traffic density, and scenario selection. No function is fixed — every parameter can be overridden or driven by real-world data.

---

## File structure

```
urban3d.html          # Single-file application (self-contained)
claude.md             # This documentation file
```

The entire simulator runs in a single HTML file. No build step, no dependencies to install. Open in any modern browser.

---

## Technology stack

| Component | Library / API | Purpose |
|---|---|---|
| 3D rendering | Three.js r128 (CDN) | Scene, lighting, geometry, shadows |
| Weather data | Open-Meteo API | Real-time climate (free, no API key) |
| Geocoding | Open-Meteo Geocoding API | Location name → lat/lon |
| GPS | Browser Geolocation API | Device position |
| Weather effects | HTML5 Canvas (overlay) | Rain, snow, fog, lightning |
| Fonts | Google Fonts (DM Mono, Syne) | UI typography |

---

## Architecture

### Rendering pipeline

```
Three.js WebGLRenderer
  └── threeScene (THREE.Scene)
        ├── Lighting: AmbientLight + HemisphereLight + DirectionalLight (sun)
        ├── Ground plane (PlaneGeometry, receiveShadow)
        ├── Road grid (PlaneGeometry overlays)
        ├── cityGroup (THREE.Group)
        │     ├── Building meshes (BoxGeometry, castShadow)
        │     ├── Window planes (PlaneGeometry, transparent)
        │     ├── Rooftop details
        │     └── Scenario elements (flyover deck, BRT bus, trees, metro entrances, towers)
        └── layerGroup (THREE.Group)
              └── Analysis layer geometry (rebuilt on toggle)

HTML Canvas (#weather-canvas, z-index 5, pointer-events none)
  └── Rain streaks, snow particles, fog overlay, lightning
```

### Data flow

```
User input (location string or GPS coords)
  → Open-Meteo Geocoding API  →  lat/lon
  → Open-Meteo Forecast API   →  current weather object (climate{})
  → Sync to manual sliders (temp, humidity, wind)
  → updateSkyFromClimate()    →  fog density, fog colour
  → updateSunLight()          →  directional light angle, colour, intensity
  → updateWeatherCanvas()     →  rain/snow/fog particle arrays
  → updateScores()            →  recalculate all 6 metrics
  → updateInsights()          →  update right panel + HUD
```

### Score calculation

All scores are integers in the range 5–99. Higher is better for walkability, solar access, and pedestrian comfort. Higher is worse for thermal stress, air quality index, and noise level.

```javascript
// Inputs
t        = manual.traffic        // 0–100
temp     = climate?.temp ?? manual.temp
h        = manual.humidity       // 0–100
fl       = manual.floors         // 1–25
scene    = current scenario ID

// Scenario flags
isFly    = scene === 'flyover'
isBRT    = scene === 'brt'
isGreen  = scene === 'park'
isMet    = scene === 'metro'
isHigh   = scene === 'highrise'

// Formulas
walk    = 80 - t*0.38 + (isGreen?22:0) + (isBRT?14:0) - (isFly?22:0) + (isMet?8:0) - fl*0.8
heat    = 25 + (temp-10)*1.3 + h*0.12 - (isGreen?22:0) - (isBRT?5:0)
air     = 15 + t*0.72 - (isBRT?18:0) - (isGreen?14:0) - (isMet?5:0)
solar   = 90 - (isHigh?38:0) - (isFly?28:0) - fl*1.2
noise   = 20 + t*0.65 - (isGreen?18:0) - (isMet?8:0)
comfort = walk*0.4 + (100-heat)*0.3 + (100-noise)*0.3

// All values clamped to [5, 99]
```

---

## Real-time climate integration

### API endpoints used

```
Geocoding:
GET https://geocoding-api.open-meteo.com/v1/search
  ?name={query}&count=1

Weather:
GET https://api.open-meteo.com/v1/forecast
  ?latitude={lat}
  &longitude={lon}
  &current=temperature_2m,relative_humidity_2m,apparent_temperature,
           wind_speed_10m,weather_code,uv_index,visibility,precipitation
  &wind_speed_unit=kmh
  &timezone=auto
```

Both APIs are free and require no authentication.

### Weather code → simulation effect mapping

| WMO code range | Condition | 3D effect | Canvas effect |
|---|---|---|---|
| 0 | Clear sky | High sun intensity, minimal fog | None |
| 1–2 | Partly cloudy | Reduced sun intensity | None |
| 3 | Overcast | Low sun, dense fog | None |
| 45–48 | Fog | `fog.density = 0.025`, grey fog colour | Semi-transparent grey fill |
| 51, 61, 80 | Rain / showers | `fog.density = 0.012`, blue-grey fog | Animated rain streaks (200 drops) |
| 71–77 | Snow | White fog tint | Drifting snow particles (120 flakes) |
| 95 | Thunderstorm | Dark fog, `fog.density = 0.018` | Rain + random lightning bolt |

### Climate object schema

```javascript
climate = {
  label:      string,   // "Berlin, Germany"
  temp:       number,   // °C, rounded
  feels:      number,   // °C apparent temperature
  humidity:   number,   // % relative humidity
  wind:       number,   // km/h
  uv:         number,   // UV index
  visibility: string,   // "8.2km"
  code:       number,   // WMO weather code
  cond:       string,   // "Partly cloudy"
  precip:     number,   // mm precipitation
}
```

---

## Scenarios

Each scenario physically modifies the `cityGroup` THREE.Group by adding tagged geometry (`userData.scenario = true`). Switching scenarios removes all tagged objects and rebuilds.

| Scenario ID | Description | 3D elements added | Score impact |
|---|---|---|---|
| `base` | Existing urban fabric | None | Baseline |
| `flyover` | Elevated road bridge | Pillars (CylinderGeometry), deck (BoxGeometry), railings | walk −22, solar −28 |
| `brt` | Bus rapid transit lane | Coloured road plane, stop markers, animated bus mesh | walk +14, air −18 |
| `park` | Green buffer zone | Cone trees (ConeGeometry), trunk cylinders, grass plane | walk +22, heat −22, air −14, noise −18 |
| `metro` | Underground metro | Station entrance boxes, signage planes, tunnel ground plane | walk +8, air −5, noise −8 |
| `highrise` | High-density towers | Tall BoxGeometry towers + transparent glass curtain wall | solar −38 |

---

## Analysis layers

Layers are rendered as Three.js geometry into `layerGroup`. The group is fully cleared and rebuilt on every toggle.

| Layer ID | Colour | 3D geometry type | What it shows |
|---|---|---|---|
| `traffic` | `#ff4f6a` | PlaneGeometry road fills + animated SphereGeometry cars | Vehicle density and flow direction |
| `heat` | `#ffb340` | PlaneGeometry ground fill + CylinderGeometry heat columns | Urban heat island concentration zones |
| `isovist` | `#4f9eff` | RingGeometry (3 rings per node) | Visibility field radii from key pedestrian nodes |
| `daylight` | `#ffd070` | SphereGeometry sun + PlaneGeometry shadow patches | Solar position and building shadow coverage |
| `air` | `#2dffaa` | SphereGeometry particles (80 instances) | Air quality dispersion, colour-coded by AQI |
| `space` | `#b06aff` | PlaneGeometry grid cells (10×10m) | Space syntax integration values |
| `noise` | `#f0997b` | RingGeometry (5 rings) | Noise propagation radii from road centreline |
| `access` | `#3dffa0` | PlaneGeometry pedestrian paths (red X if blocked) | Pedestrian route availability |

---

## Camera controls

### Viewport interaction

| Action | Effect |
|---|---|
| Click + drag | Orbit camera (sets `autoRotate = false`) |
| Scroll wheel | Zoom in/out (`camRadius` 15–150) |

### Preset buttons

| Button | `targetTheta` | `targetPhi` | `camRadius` |
|---|---|---|---|
| Auto-rotate | continuous +0.003/frame | current | current |
| Top view | 0 | 1.49 (near vertical) | 100 |
| Street level | current | 0.18 (near horizon) | 30 |
| Isometric | 0.785 (45°) | 0.785 | 90 |

Camera uses smooth lerp: `camTheta += (targetTheta - camTheta) * 0.05` per frame.

---

## Lighting system

The sun (`DirectionalLight`) position is driven by the time slider:

```javascript
const hour = (manual.time * 10 / 60) % 24
const angle = (hour - 6) / 18 * Math.PI  // 0 at 6am, π at midnight
sunX = Math.cos(angle) * 80
sunY = Math.max(5, Math.sin(angle) * 70)
sunZ = -40
```

| Time period | Sun colour | Sun intensity | Ambient intensity | Fog colour |
|---|---|---|---|---|
| Night (< 6 or > 20) | — | 0.1 | 0.2 | `#050810` |
| Dawn / dusk (6–8, 18–20) | `#ff9944` orange | 1.2 | 0.5 | `#1a1a2e` |
| Daytime (8–18) | `#fff4e0` warm white | 2.2 | 0.8 | `#0a0c14` |
| Night mode override | any | ×0.1 | 0.15 | `#020306` |

Shadow map: 2048×2048, PCFSoft, orthographic bounds ±80 units.

---

## Prompting conventions for Claude analysis

When the **Analyse ↗** button is pressed, the simulator constructs and sends this prompt:

```
3D Urban simulation analysis — Scenario: {scene3d}, Location: {climate.label},
Temperature: {temp}°C, Weather: {climate.cond}, Traffic: {manual.traffic}%,
Building floors: {manual.floors}, Humidity: {humidity}%, Wind: {wind} km/h,
Walkability: {s.walk}, Thermal stress: {s.heat}, Air quality index: {s.air},
Solar access: {s.solar}, Noise level: {s.noise}, Pedestrian comfort: {s.comfort}.
Active analysis layers: {layers}.
Please give detailed urban design recommendations.
```

### Recommended analysis workflow

1. Set a real location to pull live climate data
2. Select the scenario you want to evaluate
3. Toggle the relevant analysis layers (at minimum: traffic + the layer most relevant to your concern)
4. Adjust traffic density and building height sliders to match your site conditions
5. Press **Analyse ↗** to send the full state to Claude

### Score interpretation guide

| Score | Walkability | Thermal stress | Air quality index | Solar access |
|---|---|---|---|---|
| > 75 | Excellent pedestrian environment | Very comfortable | Clean air | Full solar access |
| 50–75 | Acceptable with improvements | Moderate discomfort | Moderate pollution | Partial shadowing |
| 25–50 | Poor, intervention needed | High stress | Poor air quality | Heavy shadowing |
| < 25 | Hostile to pedestrians | Dangerous heat | Hazardous | Near-zero access |

---

## Extending the simulator

### Adding a new scenario

1. Add a button to `#scene-grid` with `data-scene="your-id"`
2. Add a case inside `addScenarioElements()` with `userData.scenario = true` on all meshes
3. Add score adjustments to `calcScores()` with a new flag (`const isYours = scene3d === 'your-id'`)
4. Add a verdict entry to the `verdicts` object in `updateScores()`

### Adding a new analysis layer

1. Add an entry to the `layerDefs` array with `id`, `name`, `color`
2. Add `[id]: false` to the `layers` object
3. Add a draw block inside `updateLayerMeshes()` checking `if (layers.yourId)`
4. All geometry added to `layerGroup` is auto-cleared on next rebuild

### Adding a new climate variable

The Open-Meteo API supports many additional current variables. Add them to the `current=` query string:

```
&current=...,surface_pressure,cloud_cover,is_day,sunshine_duration
```

Then read them from `data.current` inside `fetchClimateCoords()` and store in the `climate` object.

---

## Known limitations

- Building geometry is procedural and does not reflect real urban morphology. For site-specific analysis, overlay a real image using the previous 2D version of the simulator.
- Score formulas are simplified linear models. They reflect relative performance across scenarios, not calibrated real-world measurements.
- The Open-Meteo API returns current conditions, not hourly forecasts. For time-of-day simulation, the time slider drives solar position only — weather does not change with time.
- Three.js r128 does not support CapsuleGeometry (introduced in r142). Cylindrical primitives are used instead.
- WebGL performance varies by device. On low-end hardware, reduce the number of active analysis layers.

---

## Acknowledgements

- Weather data: [Open-Meteo](https://open-meteo.com/) (open-source, CC BY 4.0)
- 3D engine: [Three.js](https://threejs.org/) (MIT licence)
- Urban analysis methodology references: space syntax theory (Hillier & Hanson), isovist analysis (Benedikt), UTCI thermal comfort index, WHO air quality guidelines

---

*Generated by Claude · Urban 3D Simulator v1.0*
