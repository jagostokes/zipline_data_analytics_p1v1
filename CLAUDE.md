# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A **single-page, browser-only discrete-event simulation** of a Zipline-style drone delivery logistics system. Zero backend - all simulation, scheduling, animation, and metrics run entirely in JavaScript.

Opens directly via `index.html`. No build tools, no frameworks, no server.

## Core Architecture

### HTML Structure
- `#map`: Full-screen Leaflet map (left/main area)
- `#panel`: Right sidebar with controls and metrics
  - Parameter sliders (fleet size, radius, orders/hr, speeds, times)
  - Start/Stop/Reset buttons
  - Live metrics display (avg wait, p95 wait, completed, queue size)

### Dependencies (CDN)
- Leaflet 1.9.4 for mapping
- OSM tiles for basemap
- `osmtogeojson` for building footprint parsing

### Leaflet Layers
Three layer groups managed independently:
1. `buildingsLayer`: Building footprints (static after load)
2. `ordersLayer`: Recent delivery locations (optional, capped at ~200 for performance)
3. `dronesLayer`: Drone markers (always rendered, animated)

## Simulation Model

### Coordinate System & Distance
- **Hub**: Fixed at San Francisco (37.7749, -122.4194)
- **Max Range**: 8km hard constraint (Zipline operational limit)
- **Distance**: Equirectangular approximation for small distances
  ```javascript
  distanceKm(lat1, lon1, lat2, lon2) {
    const R = 6371; // Earth radius in km
    const x = (lon2 - lon1) * Math.cos((lat1 + lat2) / 2);
    const y = lat2 - lat1;
    return Math.sqrt(x*x + y*y) * R;
  }
  ```
- **Range Enforcement**: Orders beyond 8km are rejected and skipped

### Order Generation: Uniform-in-Disk Sampling
Generate orders with uniform spatial density within delivery radius:
```javascript
// Sample radius: r = R * sqrt(u) to get uniform area distribution
// Sample angle: theta = 2π * u
// Use R = 1.5 * effectiveRadius so E[r] ≈ effectiveRadius
// effectiveRadius = min(avgRadiusKm, MAX_RANGE_KM) enforces 8km limit
const effectiveRadius = Math.min(avgRadiusKm, MAX_RANGE_KM);
const r = 1.5 * effectiveRadius * Math.sqrt(Math.random());
const theta = 2 * Math.PI * Math.random();
// Convert to lat/lng offset (small-distance approximation)
// Orders exceeding MAX_RANGE_KM are rejected and skipped
```

### Order Arrival: Deterministic Per-Second Accumulator
Avoids Poisson complexity for MVP:
```javascript
// Each simulation second:
orderAccumulator += ordersPerHour / 3600;
while (orderAccumulator >= 1.0) {
  spawnOrder(simT);
  orderAccumulator -= 1.0;
}
```

### Timing Model (The Only Physics)
```javascript
const flyTimeOneSec = (distanceKm / speedKmh) * 3600;
const busyTimeTotal = loadSec + flyTimeOneSec + serviceAtDropSec + flyTimeOneSec + turnaroundSec;
```

Where:
- `loadSec`: Time to load package at hub
- `flyTimeOneSec`: One-way flight time
- `serviceAtDropSec`: Time spent at delivery location
- `turnaroundSec`: Reload/recharge time at hub

### Drone Scheduling: Greedy Earliest-Available
```javascript
// On new order or drone freed:
const drone = findDroneWithMinAvailableAt(drones);
const tStart = Math.max(order.tCreated, drone.availableAt);
order.waitSec = tStart - order.tCreated;
order.tStart = tStart;
order.tFinish = tStart + busyTimeTotal;
drone.availableAt = order.tFinish;
```

### Drone State Structure
```javascript
{
  id: number,
  availableAt: number,        // simT when drone is free
  queue: [segment, ...],      // Animation segments to render
  active: segment | null,     // Current segment being animated
  marker: L.Marker,           // Leaflet marker
  lat: number,                // Current position
  lng: number
}
```

### Animation Segments
Each assigned order generates two segments pushed to `drone.queue`:
```javascript
// Segment 1: Hub → Delivery (starts after load time)
{
  t0: tStart + loadSec,
  t1: tStart + loadSec + flyTimeOneSec,
  fromLat: hubLat,
  fromLng: hubLng,
  toLat: order.lat,
  toLng: order.lng
}

// Segment 2: Delivery → Hub (starts after service time)
{
  t0: t1 + serviceAtDropSec,
  t1: t1 + serviceAtDropSec + flyTimeOneSec,
  fromLat: order.lat,
  fromLng: order.lng,
  toLat: hubLat,
  toLng: hubLng
}
```

### Animation Loop (RAF)
```javascript
function animate(timestamp) {
  if (!running) return;

  const dtReal = (timestamp - lastFrameTime) / 1000;
  lastFrameTime = timestamp;
  simT += dtReal * timeScale;

  // For each drone:
  // 1. If active === null && queue.length > 0: active = queue.shift()
  // 2. If simT < active.t0: hold position (loading/servicing)
  // 3. Else: alpha = clamp((simT - active.t0) / (active.t1 - active.t0))
  //          lat/lng = lerp(from, to, alpha)
  // 4. If simT >= active.t1: snap to destination, active = null

  updateDroneMarkers();
  requestAnimationFrame(animate);
}
```

## Metrics Calculation

### Streaming Statistics
- **Completed orders**: Increment when `simT >= order.tFinish`
- **Average wait**: Incremental mean update
  ```javascript
  avgWait = (avgWait * (n - 1) + newWait) / n
  ```

### P95 Wait Time: Bounded Reservoir
Maintain a ring buffer of last N wait times (e.g., 2000):
```javascript
// On order completion:
waitBuffer[completedCount % BUFFER_SIZE] = order.waitSec;

// Compute p95 (every 250ms, not every frame):
const sorted = [...waitBuffer].sort((a, b) => a - b);
const p95 = sorted[Math.floor(sorted.length * 0.95)];
```

### Completion Tracking: Min-Heap
For accurate real-time metrics, maintain a priority queue of pending completions:
```javascript
// When order scheduled:
pendingCompletions.push({ tFinish: order.tFinish, order });

// Each sim tick:
while (pendingCompletions.length && pendingCompletions[0].tFinish <= simT) {
  const { order } = pendingCompletions.shift();
  updateMetrics(order.waitSec);
}
```

## Building Footprints (Overpass API)

### Query Construction
```javascript
const bbox = computeBbox(hubLat, hubLng, buildingsRadiusKm);
const query = `
[out:json];
(
  way["building"](${bbox.south},${bbox.west},${bbox.north},${bbox.east});
  relation["building"](${bbox.south},${bbox.west},${bbox.north},${bbox.east});
);
out geom;
`;
```

### Fetch and Render
```javascript
// POST to https://overpass-api.de/api/interpreter
// Parse with osmtogeojson
// Render as L.geoJSON with low opacity (0.2-0.3)
// Cache in memory; on failure, keep app functional without buildings
```

### Guardrails
- Use separate `buildingsRadiusKm` parameter (can differ from delivery radius)
- Implement timeout and fallback for Overpass failures
- Clear and re-fetch only when bbox changes significantly

## Control Flow

### Reset
```javascript
function reset() {
  simT = 0;
  running = false;
  orderAccumulator = 0;
  orders = [];
  completedOrders = [];
  pendingCompletions = [];

  // Clear layers
  ordersLayer.clearLayers();
  dronesLayer.clearLayers();

  // Recreate drones
  drones = Array.from({ length: params.numDrones }, (_, i) => ({
    id: i,
    availableAt: 0,
    queue: [],
    active: null,
    marker: createDroneMarker(hubLat, hubLng),
    lat: hubLat,
    lng: hubLng
  }));

  // Add markers to layer
  drones.forEach(d => d.marker.addTo(dronesLayer));

  // Reset metrics display
  updateMetricsUI();
}
```

### Start
```javascript
function start() {
  running = true;
  lastFrameTime = performance.now();
  requestAnimationFrame(animate);
}
```

### Stop
```javascript
function stop() {
  running = false;
  // Simulation clock freezes, but state is preserved
}
```

## Performance Considerations

### Rendering Caps
- **Drone markers**: Always render (typically < 50)
- **Order markers**: Cap at 200 most recent, or omit entirely
- **Buildings**: Render once, cache, use low detail
- **Metrics update**: Every 250ms, not every frame

### Simulation vs. Visualization Decoupling
- `timeScale` parameter allows fast-forward
- Simulation step (event processing) can run faster than 60fps rendering
- High `ordersPerHour` (100+) requires efficient event queue

### Memory Management
- Use ring buffers for wait time statistics (fixed size)
- Clear old order markers beyond cap
- Cache building GeoJSON, don't re-parse on every reset

## Optional: Fleet Optimizer ("Find N")

Binary search for minimum fleet size to meet SLA:
```javascript
function findMinDrones(targetP95Sec, horizonSec) {
  let lo = 1, hi = 100;
  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2);
    const p95 = runOfflineSim(mid, horizonSec); // No animation
    if (p95 <= targetP95Sec) {
      hi = mid;
    } else {
      lo = mid + 1;
    }
  }
  return lo;
}
```

Offline simulation:
- No RAF loop, no marker updates
- Pure event scheduling and metrics computation
- Run for 2-6 simulated hours to get stable p95

## Default Parameters (Demo Preset)

```javascript
const DEMO_PRESET = {
  hubLat: 37.7749,
  hubLng: -122.4194,
  avgRadiusKm: 2,
  buildingsRadiusKm: 3,
  numDrones: 10,
  speedKmh: 70,
  loadSec: 30,
  serviceAtDropSec: 30,
  turnaroundSec: 60,
  ordersPerHour: 60,
  timeScale: 1.0
};
```

## Development Notes

### Testing Changes
1. Edit source, refresh browser (hard refresh: Cmd+Shift+R / Ctrl+Shift+R)
2. Open console for errors
3. Test with demo preset first, then stress test (50 drones, 100 orders/hr)

### Common Gotchas
- **Time units**: Keep `simT` in seconds, convert to/from hours only for display
- **Distance units**: Internal km, convert to meters for Leaflet if needed
- **Angle units**: Radians for math, degrees for Leaflet
- **Animation gaps**: Load and service times are "hold position" gaps, not segments
- **Completion timing**: Order is "complete" when `simT >= tFinish`, not when animation ends

### Code Organization
- Single `index.html` with embedded `<script type="module">`
- Can refactor to separate `js/` modules if complexity grows
- Keep simulation logic (pure functions) separate from DOM/Leaflet manipulation

## Mathematical Correctness

This simulation is **physically plausible at city scale**:
- Uniform-in-disk sampling ensures proper spatial density
- Equirectangular distance is accurate within ~1% for distances < 50km at mid-latitudes
- Greedy scheduling is optimal for identical drones (no benefit to reordering)
- p95 convergence requires ~100-200 samples minimum
