# SF Zipline Simulation

A real-time, browser-based discrete-event simulation of a Zipline-style drone delivery logistics system for San Francisco. Watch autonomous drones fulfill delivery orders across real SF neighborhoods with live performance metrics.

![SF Zipline Simulation](https://img.shields.io/badge/status-active-brightgreen) ![License](https://img.shields.io/badge/license-MIT-blue)

## 🚀 Features

- **Real-time drone animation** - Watch drones fly between the hub and delivery locations with smooth path interpolation
- **Real building locations** - Orders are delivered to actual residential buildings in SF (Richmond, Sunset, Bayview, Mission, Western Addition)
- **Live performance metrics** - Track average delivery time, P95 wait time, completion rate, and system throughput
- **Dynamic simulation controls** - Adjust fleet size (up to 1000 drones), delivery radius, order rate (up to 1000/hr), and time scale (up to 50x) on the fly
- **Dark themed UI** - Professional dashboard with floating controls and metrics
- **Zero backend** - Runs entirely in the browser using JavaScript and Leaflet.js

## 📋 Prerequisites

- Modern web browser (Chrome, Firefox, Safari, Edge)
- Internet connection (for loading map tiles and building data)
- Optional: Local web server for best performance

## 🏃 Quick Start

### Option 1: Direct File Open
1. Download `index.html`
2. Open it directly in your browser
3. Click "Load SF Residential Districts"
4. Adjust parameters as desired
5. Click **▶ START**

### Option 2: Local Server (Recommended)
```bash
# Using Python 3
python3 -m http.server 8000

# Using Node.js
npx http-server

# Then open: http://localhost:8000/index.html
```

## 🎮 Controls

### Floating Controls (Right Side)
- **▶ START** - Begin the simulation
- **■ STOP** - Pause the simulation (preserves state)
- **⟲ RESET** - Reset simulation to initial state

### Performance Metrics (Left Side)
Real-time dashboard showing:
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

## 🎯 Use Cases

### 1. Fleet Sizing
Determine the minimum number of drones needed to maintain service level agreements:
- Set target order rate (e.g., 500/hr)
- Adjust fleet size until P95 wait time < target (e.g., 10 minutes)
- Use high time scale (20x-50x) to quickly reach steady state

### 2. Service Area Analysis
Understand the relationship between delivery radius and fleet requirements:
- Start with small radius (1-2 km)
- Gradually increase and observe impact on delivery times
- Note the quadratic growth in area vs. linear growth in drone travel time

### 3. Capacity Planning
Stress test the system with extreme demand:
- Set orders/hr to 1000
- Set fleet size to 500-1000
- Run at 50x time scale to simulate hours of operation in minutes

### 4. Parameter Sensitivity
Explore how operational parameters affect performance:
- Reduce turnaround time → Higher effective fleet size
- Increase drone speed → Shorter delivery times, lower wait times
- Increase service time → More drones needed for same throughput

## 🛠️ Technical Details

### Architecture
- **Single-page application** - No build tools, frameworks, or server required
- **Discrete-event simulation** - Mathematically accurate event scheduling
- **Poisson arrivals** - Realistic stochastic order generation
- **Greedy scheduling** - Optimal earliest-available drone assignment
- **60 FPS animation** - Smooth drone movement with linear interpolation

### Data Sources
- **Map tiles**: OpenStreetMap via CartoDB Dark Matter
- **Building data**: OpenStreetMap Overpass API
- **Coordinate system**: WGS84 (latitude/longitude)
- **Distance calculation**: Equirectangular approximation (accurate to ~1% for <50km)

### Performance Optimizations
- Pre-filtered building locations by delivery radius (O(1) sampling)
- Capped order marker rendering (max 200 visible)
- Incremental statistics (streaming mean, ring buffer for P95)
- Efficient drone state management (queue-based animation segments)

### Simulation Model
Each delivery follows this timeline:
1. **Order Created** (t=0) - Customer places order
2. **Wait Period** - Order waits for available drone
3. **Loading** (t=tAssign) - Package loaded at hub
4. **Outbound Flight** (t=tAssign + loadTime) - Drone flies to customer
5. **Service** (t=arrival) - Package delivered
6. **Return Flight** (t=arrival + serviceTime) - Drone returns to hub
7. **Turnaround** (t=return) - Drone recharges/reloads
8. **Available** (t=return + turnaroundTime) - Drone ready for next order

## 📊 Metrics Explained

### Average Delivery Time
Time from order placement to successful delivery (order → drop-off complete). Includes wait time, loading, flight, and service.

### P95 Wait Time
95th percentile of time orders spend waiting for drone assignment. A key SLA metric - means 95% of orders wait less than this time.

### Actual Rate
Measured order generation rate in simulation time. Should match the "Orders per Hour" slider. If lower, check for performance issues or slider value.

### Effective Speed
Ratio of simulation time advancement to real time. At steady state, should match Time Scale slider. Lower values indicate performance bottleneck.

## 🐛 Troubleshooting

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
