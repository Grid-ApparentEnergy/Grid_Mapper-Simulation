# Function Documentation - Simulation Page

This document describes the important functions in the SimulationPage component.

---

## SimulationPage Core Functions

### `handleEnergize()`

Energizes the grid using BFS (Breadth-First Search) from all power sources.

**Logic:**
1. Validates that adjacency list and grid data exist
2. Checks if power sources are identified
3. Calls `getEnergizedStatus()` with all combined sources to ensure the entire grid can logically energize
4. Counts live buses from the energized status map
5. Updates simulation state with new energized status
6. If sensors are placed, updates their readings based on new energized status
7. Stores initial energized status for future fault comparisons (to gracefully hide dead-from-start wires)
8. Displays performance metrics (time elapsed, buses energized)

---

### `handleDeenergize()`

De-energizes the entire grid and clears fault information.

**Logic:**
1. Resets energized status to null
2. Clears fault information
3. Clears sensor readings
4. Updates energized flag to false
5. Shows toast notification

---

### `handlePlaceSensors()`

Places sensors at strategic intervals along the energized power grid using pole locations.

**Logic:**
1. Validates grid is energized before placing sensors
2. Builds bus geography map (bus ID → [lon, lat])
3. Uses poles as potential sensor locations
4. Calls `placeSensorsIntervalBased()` with:
   - Pole locations
   - Bus geography map
   - Sensor interval (L)
   - Adjacency list for DFS traversal
   - Power sources
   - Substations to define exclusion zones
   - Initial Energized Status (to filter paths immediately during traversal)
5. The `placeSensorsIntervalBased` engine executes a recursive DFS strategy:
   - Places a sensor at the START of each energized path (just outside a 300m radius from the substation).
   - Places sensors exactly every L poles along the path.
   - Places a sensor at the END of each energized path.
   - Ignores nodes within 300m of a substation to prevent redundant coverage.
6. Filters out sensors on non-energized buses
7. Calculates sensor readings for energized sensors
8. Updates simulation state with sensor locations, intervals, and metrics
9. Displays placement statistics

---

### `handleTriggerFault(lineIdx)`

Triggers a fault on a specific transmission line (click-to-fault only).

**Logic:**
1. Validates adjacency list and grid data exist
2. Requires lineIdx parameter (no random faults)
3. Finds the line object from grid data
4. Creates disabled lines set containing only the faulted line
5. Recalculates energized status with the disabled line
6. Creates fault info object with line details (index, from/to buses, voltage)
7. Counts dead buses after fault
8. If sensors are placed:
   - Updates sensor readings based on new energized status
   - Identifies faulty interval using `identifyFaultyInterval()`
9. Updates simulation state with new energized status and fault info
10. Displays fault impact metrics (line index, dead buses, time elapsed)

---

### `handleRepairFault()`

Repairs the active fault and restores grid to normal operation.

**Logic:**
1. Validates adjacency list and grid data exist
2. Recalculates energized status with no disabled lines (empty set)
3. Updates simulation state:
   - Sets energized flag to true
   - Updates energized status to fault-free state
   - Clears fault information
   - Resets faulty block index
   - Updates sensor readings if sensors are placed
4. Displays repair confirmation with time elapsed

---

### `handleReset()`

Resets the entire simulation to initial state.

**Logic:**
1. Resets simulation state to `INITIAL_SIM_STATE`:
   - energized: false
   - sensors: []
   - intervals: []
   - sensorMetrics: null
   - sensorReadings: null
   - energizedStatus: null 
   - initialEnergizedStatus: null
   - faultInfo: null
   - faultyInterval: -1
   - repairMode: false
2. Disables fault isolation mode
3. Shows reset confirmation toast

---

### `loadGridData(regionBounds)`

Loads grid data from the backend API for a specific geographic region.

**Logic:**
1. Determines API URL (from environment variable or default)
2. Builds query string from region bounds if provided:
   - min_lon, min_lat, max_lon, max_lat
3. Fetches grid data from `/api/grid-data` endpoint
4. On success:
   - Updates grid data state
   - Builds adjacency list using `buildAdjacencyList()`
   - Extracts all bus IDs
   - Sets power sources from substation_sources
   - Displays load confirmation with statistics
5. On error:
   - Logs error to console
   - Shows error toast with details

---

### `handleToggleAreaSelection()`

Toggles the area selection mode on the map.

**Logic:**
1. Toggles `isSelectingArea` state
2. When enabled, map enters rectangle drawing mode
3. When disabled, exits drawing mode

---

### `handleAreaSelected(bounds)`

Handles completion of area selection on the map.

**Logic:**
1. Receives bounds object with min_lon, min_lat, max_lon, max_lat
2. Updates `selectedAreaBounds` state
3. Disables area selection mode
4. Shows toast notification
5. Triggers `useEffect` to reload grid data for selected area

---

### `handleSelectionCancel()`

Cancels the area selection process.

**Logic:**
1. Sets `isSelectingArea` to false
2. Exits rectangle drawing mode without saving bounds

---

### `useEffect` - Area Change Handler

Reloads grid data when the selected area changes.

**Logic:**
1. Watches `selectedAreaBounds` for changes
2. When bounds change:
   - Resets simulation state to initial
   - Disables fault isolation
   - Calls `loadGridData()` with new bounds
3. Dependencies: selectedAreaBounds, loadGridData

---

## Key State Variables

- **gridData**: Contains buses, lines, towers, poles, substations from API
- **simState**: Simulation state including energized status, sensors, faults
- **adjRef**: Reference to adjacency list (graph representation)
- **allBusesRef**: Reference to array of all bus IDs
- **sources**: Array of power source bus IDs (substations)
- **selectedAreaBounds**: Geographic bounds of selected area
- **sensorInterval**: Interval between sensors (L = 50 poles default)

---

## SensorPredictorPage Core Functions

### `verifyCoverage(traversableNodes, totalSensors, intervalL, ruleCounts)`

Verifies that every traversable node is within L hops of at least one sensor.

**Logic:**
1. Uses a worst-case analytical bound (capacity = totalSensors * intervalL).
2. For non-tree (cyclic) graphs, actual coverage is strictly better, making this bound safely conservative.
3. Generates a detailed breakdown report string for the user indicating PASS/FAIL status.

---

### `computeWeights(regionKey)`

Computes the final normalized slice of hazards for the pie chart.

**Logic:**
1. Starts with `BASE_WEIGHTS` (wind, lightning, flood, etc.).
2. Looks up regional modifiers (e.g., Kutch gets +0.15 wind).
3. Sums baseline + modifiers.
4. Normalizes all weights so they sum exactly exactly to 1.0 (100%).

---

### `fetchWeatherAndComputeRisk(regionKey)`

Fetches live 72-hour forecast from Open-Meteo and computes a granular hourly risk score.

**Logic:**
1. Fires request to Open-Meteo API using the region's lat/lon bounds for a 3-day hourly forecast.
2. Extracts variables: precipitation, windspeed, windgusts, pressure, relative humidity, cloud cover.
3. Applies proxy feature-engineering formulas (e.g. `wind_stress`, `lightning_risk`, `cyclone_risk`).
4. Suppresses false positive baseline variances.
5. Computes the composite index via weighted average across the 72 hours.
6. Returns an aggregate `finalRisk` (0.0 to 1.0) and the raw data payload.

---

### `calculateSensors()`

The core algorithmic estimation engine that builds the predicted topology.

**Logic:**
1. Generates theoretical totals for Substation (N_sub) and Dead-end nodes (N_dead).
2. Calculates overlapping sensor counts based on R1 (Feeder Exit) and R3 (Dead-ends).
3. Applies the Real-Time weather risk to dynamically shrink the default interval size $L$. E.g., high risk decreases $L$ from 50 hops to 30 hops to increase sensor density.
4. Computes R2 (DFS Interval) requirements.
5. Deduplicates the overlaps between (R1+R3) against R2.

---

## InfrastructurePlannerPage Core Functions

### `handleDrawCreated(e)`

Handles map click-and-drag interactions for drawing custom grid Topologies.

**Logic:**
1. Separates events strictly into markers (Substations) and polylines (Transmission Lines).
2. Implements a greedy geographic SNAP threshold (0.05 lat/lon degree).
3. If an endpoint is dropped too close to an existing pole, it perfectly snaps the edge to share the existing vertex to ensure graph connectivity.

---

### `handleNodeClick(node)`

Handles direct post-generation mutation of drawn/clipped grid infrastructure.

**Logic:**
1. Respects the active `nodeClickMode` radio toggle.
2. Mode "substation": Converts standard pole topologies into substations (power sources) and vice-versa.
3. Mode "sensor": Injects manual overrides to forcibly place or remove sensors on specific distinct towers.

---

### `processFile(file)`

Parses user-dropped infrastructure files into React graph state.

**Logic:**
1. Detects extension via `.name`.
2. Proxies GeoJSON `.json` shapes to `parseGeoJSON()`, pulling out Linestrings strictly.
3. Proxies `.csv` node/edge link tables to `parseCSV()`.

---

### `handleFetchDB(e)`

Hits the backend Postgres DB with an arbitrary geographic bounding box to perfectly clone real-world segments.

**Logic:**
1. Assembles query strings from `bounds` hook (`min_lon`, `min_lat`, etc.).
2. Retrieves matching segments from `/api/grid-data`.
3. Isolates `substation_sources` to upgrade standalone DB buses into full substation objects natively.
4. Directly appends the clipped graph to the local graph store.

---

### `handleRunPipeline()`

Final execution function to deploy sensors onto the assembled graph structure.

**Logic:**
1. Validates graph topology.
2. Dispatches local Nodes, Edges, and the desired Sensor Interval L to `plannerEngine.js`.
3. Computes bounding box metrics and the absolute elapsed compute time in ms.
