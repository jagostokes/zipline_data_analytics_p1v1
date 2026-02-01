# SF Zipline Simulation

A browser-based discrete-event simulation of drone delivery logistics for San Francisco. The entire simulation runs client-side - no backend, no build tools, just open the HTML file and watch drones fulfill delivery orders across actual SF neighborhoods.

## Features

- Real-time drone animation with smooth interpolation between hub and delivery locations
- Orders delivered to actual residential buildings in SF (Richmond, Sunset, Bayview, Mission, Western Addition)
- Hard 8km delivery range constraint (Zipline operational limit - orders beyond this are rejected)
- Live metrics: average delivery time, P95 wait time, completion rate, throughput
- Adjustable parameters: fleet size (up to 1000), delivery radius, order rate (up to 1000/hr), time scale (up to 50x)
- Dark themed UI with floating controls
- Runs entirely in the browser - JavaScript + Leaflet.js, zero backend

## Prerequisites

- Modern browser (Chrome, Firefox, Safari, Edge)
- Internet connection for map tiles and building data
- Optional: local web server for better performance

## Quick Start

### Option 1: Direct File Open
1. Download `index.html`
2. Open it in your browser
3. Click "Load SF Residential Districts"
4. Adjust parameters
5. Click START

### Option 2: Local Server (Recommended)
```bash
# Python 3
python3 -m http.server 8000

# Node.js
npx http-server

# Open http://localhost:8000/index.html
```

## Controls

### Floating Controls (Right Side)
- START - Begin the simulation
- STOP - Pause (preserves state)
- RESET - Reset to initial state

### Performance Metrics (Left Side)
- **Avg Delivery** - Average time from order placement to delivery completion
- **P95 Wait** - 95th percentile wait time before drone assignment
- **Completed** - Total orders fulfilled
- **Queue Size** - Orders waiting for drone assignment
- **Active Drones** - Drones currently on delivery missions
- **Total Orders** - Orders generated since simulation start
- **Rate (Actual)** - Actual order generation rate (orders/hour)
- **Sim Time** - Elapsed simulation time
- **Real Time** - Elapsed real-world time
- **Speed** - Current simulation speed multiplier

### Parameter Sliders (Right Panel)

#### Environment
- **Delivery Radius** (0.5-10 km) - Maximum delivery distance from hub
- **Buildings Radius** (1-20 km) - Area to load building data from

#### Fleet Configuration
- **Number of Drones** (1-1000) - Total drone fleet size
- **Drone Speed** (30-120 km/h) - Cruise speed of drones

#### Timing Parameters
- **Load Time** (10-120 sec) - Package loading time at hub
- **Service Time** (10-120 sec) - Time spent at delivery location
- **Turnaround Time** (30-300 sec) - Reload/recharge time at hub

#### Demand & Speed
- **Orders per Hour** (1-1000) - Order arrival rate
- **Time Scale** (0.1x-50x) - Simulation speed multiplier

#### Preset & Districts
- **Load Demo Preset** - Reset to default balanced parameters
- **Load SF Residential Districts** - Fetch real building locations from OpenStreetMap

## Use Cases

### Fleet Sizing
Find the minimum number of drones to maintain service levels:
- Set target order rate (e.g., 500/hr)
- Adjust fleet size until P95 wait time hits target (e.g., < 10 minutes)
- Run at 20x-50x speed to reach steady state quickly

### Service Area Analysis
See how delivery radius affects fleet requirements:
- Start with small radius (1-2 km)
- Gradually increase and watch delivery times
- Area grows quadratically, drone travel time grows linearly

### Capacity Planning
Stress test with extreme demand:
- Set orders/hr to 1000
- Set fleet size to 500-1000
- Run at 50x speed to simulate hours of operation in minutes

### Parameter Sensitivity
See how operational parameters affect performance:
- Reduce turnaround time = higher effective fleet size
- Increase drone speed = shorter delivery times, lower wait times
- Increase service time = more drones needed for same throughput

## Technical Details

### Architecture
- Single-page app - no build tools, frameworks, or server
- Discrete-event simulation with accurate event scheduling
- Poisson arrivals for realistic stochastic order generation
- Greedy scheduling (earliest-available drone assignment)
- Hard 8km maximum delivery range (Zipline operational limit)
- 60 FPS animation with linear interpolation

### Data Sources
- Map tiles: OpenStreetMap via CartoDB Dark Matter
- Building data: OpenStreetMap Overpass API
- Coordinate system: WGS84 (lat/lng)
- Distance: Equirectangular approximation (accurate to ~1% for <50km)

### Performance Optimizations
- Pre-filtered building locations by delivery radius (O(1) sampling)
- Capped order marker rendering (max 200 visible)
- Incremental statistics (streaming mean, ring buffer for P95)
- Queue-based animation segments for efficient drone state management

### Simulation Model
Delivery timeline:
1. Order Created (t=0) - Customer places order
2. Wait Period - Order waits for available drone
3. Loading (t=tAssign) - Package loaded at hub
4. Outbound Flight (t=tAssign + loadTime) - Drone flies to customer
5. Service (t=arrival) - Package delivered
6. Return Flight (t=arrival + serviceTime) - Drone returns to hub
7. Turnaround (t=return) - Drone recharges/reloads
8. Available (t=return + turnaroundTime) - Drone ready for next order

## Metrics Explained

### Average Delivery Time
Time from order placement to successful delivery. Includes wait time, loading, flight, and service.

### P95 Wait Time
95th percentile of time orders spend waiting for drone assignment. Key SLA metric - 95% of orders wait less than this time.

### Actual Rate
Measured order generation rate in simulation time. Should match the "Orders per Hour" slider. If lower, you have performance issues.

### Effective Speed
Ratio of simulation time to real time. Should match Time Scale slider at steady state. Lower values mean performance bottleneck.

## Troubleshooting

### Buildings won't load
- Check browser console (F12) for errors
- Try reducing Buildings Radius to 5-10 km
- Overpass API may be rate-limited; wait 1-2 minutes and retry

### Simulation runs slowly at high parameters
- Reduce Time Scale to 5x-10x
- Reduce Orders per Hour to <500/hr
- Reduce fleet size to <200 drones
- Close other browser tabs

### Order rate drops below target
- Check console for "Could not find building within radius" warnings
- Increase Buildings Radius or Delivery Radius
- Ensure "Load SF Residential Districts" completed successfully

### Drones stuck at hub
- Click RESET to clear state
- Verify Orders per Hour > 0
- Check console for JavaScript errors

## 🤝 Contributing

This is an educational/research project. Feel free to fork and extend:
- Add multiple hubs (multi-depot routing)
- Implement battery constraints (limited flight time)
- Add wind/weather effects
- Optimize scheduling algorithm (Hungarian, auction-based)
- Export metrics to CSV for analysis
- Add real-time traffic heat maps

## 📄 License

MIT License - Feel free to use for education, research, or commercial purposes.

## 🙏 Acknowledgments

- **Leaflet.js** - Interactive map rendering
- **OpenStreetMap** - Map tiles and building data
- **osmtogeojson** - OSM data parsing
- Inspired by [Zipline](https://www.flyzipline.com/) drone delivery operations

## 📞 Support

For issues or questions, please check the browser console (F12) first. Most issues are related to:
1. Building data not loading (API rate limits)
2. Browser performance (reduce parameters)
3. Slider values not taking effect (click RESET after changing)

---

**Built with ❤️ for logistics optimization and urban air mobility research**
