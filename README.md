# Distance & Fuel Calculator

This repository contains a Streamlit application that calculates driving distance between two user-specified locations, estimates fuel requirements for a selected truck, suggests fueling stops along the route using a local LovesLocations dataset, stores a trip record in the database, and generates an interactive HTML route map saved under `generated_maps/`.

---

## Quick Overview

- Primary UI: Streamlit app (`Distance-Fuel_calculator.py`) — run with `streamlit run Distance-Fuel_calculator.py`.
- Geocoding & routing: OpenRouteService (requires `API_KEY`).
- Database: SQL Server accessed via `pyodbc` ODBC Driver (expects `ODBC Driver 17 for SQL Server`).
- Persisted artifacts: Trip records in the `Trip_Records` table and route map HTML files in `generated_maps/`.

---

## Environment Variables

- `API_KEY` — OpenRouteService API key (required for geocoding and routing).
- `DATABASE_URL` — optional connection URL used to parse a default database name. If not provided, the code will fallback to local SQL Server instances (`DESKTOP-...` or `localhost`).

The app loads variables from a `.env` file located next to `Distance-Fuel_calculator.py`.

---

## Database Expectations (required tables & columns)

The app inspects the database schema dynamically, but it expects the following logical tables/columns:

- Truck details table: `Truck_Detail` or `Truck_Details` (schema may vary)
  - A column containing the truck display name (candidates: `truck_name`, `truckname`, `name`, `truck`, `vehicle_name`, etc.)
  - A numeric column for fuel consumption expressed as liters per 100 km (candidates: `fuel_average_l`, `fuel_avg_l`, `fuel_average`, `fuel_avg`, etc.)

- Loves locations table: `dbo.LovesLocations`
  - Columns: `City`, `Address`, `Latitude`, `Longitude` (Latitude/Longitude numeric)

- Trip records table: `Trip_Records`
  - Used for inserting trip summary records. Expected columns: `Starting_point`, `Destination_point`, `Total_Distance`, `Truck_Name`, `Truck_Average_L`, `Required_Time`, `Date`.

If these tables or columns aren't present, the app attempts to detect reasonable candidates or returns readable error messages.

---

## Streamlit UI — Inputs and Validation

When you open the app (`streamlit run Distance-Fuel_calculator.py`) the UI exposes the following controls:

- Starting Point (text input)
  - Accepts: place names, "City, Address" strings, or freeform address text.
  - Validated: must be non-empty when calculating.

- Destination (text input)
  - Accepts same formats as starting point.
  - Validated: must be non-empty when calculating.

- Truck Selection (selectbox)
  - Populated from the Truck_Detail(s) table.
  - Validated: user must select one truck before calculating.

- Current Fuel in Truck (number input, litres)
  - Default: `0.00`
  - Used to compute available travel range and recommended/refuel warnings.

- Calculate Button (`📍 Calculate Distance`)
  - Triggers the main flow: geocoding → route computation → fuel management → store record → map generation.

Client-side validations will show error messages if required fields are missing.

---

## What the App Does (high-level flow)

1. Load available trucks from DB (`get_available_trucks`).
2. Read user inputs (starting point, destination, selected truck, current fuel).
3. Resolve both locations to coordinates using OpenRouteService; fallback to local `LovesLocations` DB search if ORS fails (`resolve_location_to_coordinates`).
4. Calculate driving route and metrics (`calculate_distance_with_api`) via OpenRouteService directions API.
5. Extract addresses along the route from `LovesLocations` within a corridor and mark the best fueling stop based on current fuel and truck efficiency (`_build_fuel_management_summary`).
6. Store a `Trip_Records` row (`store_trip_record`) with the summary of the trip.
7. Generate a local HTML map file in `generated_maps/` and open it in a new browser tab.
8. Present metrics and detailed info in the Streamlit UI.

---

## Programmatic API (functions you can call directly)

Although the app is driven by the Streamlit UI, these core functions are exposed in `Distance-Fuel_calculator.py` and can be called from other code/tests:

- `get_available_trucks()` → dict { success: bool, trucks: [str], error?: str }
- `get_truck_fuel_average(truck_name)` → dict { success, truck_name, fuel_average_l, error? }
- `resolve_location_to_coordinates(location_name, city=None, address=None)` → dict { success, found, latitude, longitude, label, source, error? }
- `calculate_distance_with_api(lat1, lon1, lat2, lon2)` → dict { success, distance_km, distance_miles, distance_meters, duration_seconds, route_coordinates, addresses_along_route, ... }
- `calculate_and_store_trip_record(starting_point, destination_point, truck_name, current_fuel_l=None, starting_city=None, starting_address=None, destination_city=None, destination_address=None)`
  - Returns a comprehensive dict (see example below) and also persists a `Trip_Records` entry on success.

### Example: `calculate_and_store_trip_record` return structure (trimmed sample)

```json
{
  "success": true,
  "starting_point": "Calgary, AB",
  "destination_point": "Edmonton, AB",
  "truck": { "success": true, "truck_name": "TruckA", "fuel_average_l": 30.0 },
  "start_coordinates": { "latitude": 51.0447, "longitude": -114.0719, "found": true },
  "destination_coordinates": { "latitude": 53.5461, "longitude": -113.4938, "found": true },
  "distance": {
    "success": true,
    "distance_km": 299.5,
    "distance_miles": 186.2,
    "distance_meters": 299500,
    "duration_seconds": 13500,
    "route_coordinates": [{"lat":51.0,"lon":-114.0}, ...],
    "addresses_along_route": [ {"city":"Red Deer","address":"Some Rd","latitude":52.2681,"longitude":-113.8112,"distance_from_start_km":150.3, ...} ]
  },
  "fuel_required_l": 9.98,
  "estimated_duration_hours": 3.75,
  "current_fuel_l": 40.0,
  "fuel_management": {
    "current_fuel_l": 40.0,
    "fuel_average_l_per_100km": 30.0,
    "available_travel_range_km": 133.33,
    "safe_operating_range_km": 113.33,
    "recommended_fueling_point": { "city": "Red Deer", "address": "Some Rd", "recommended_fueling_point": true, "distance_from_start_km": 150.3, "fuel_remaining_at_stop_l": 0.0, ... }
  },
  "trip_record_id": 123,
  "trip_record_label": "T123",
  "calculation_date": "06/03/2026",
  "start_location": "Calgary, AB",
  "destination_location": "Edmonton, AB"
}
```

---

## Generated Artifacts

- HTML route maps are written into `generated_maps/` with filenames like `route_T123_ab12cd34.html`, where `T123` is the stored trip record label or ID. The function `_build_route_map_file_uri()` returns a `file://` URI to the generated HTML and the app attempts to open it in a new browser tab.

---

## Error Handling & Messages

- If `API_KEY` is missing, geocoding and routing functions return an explicit error: `API_KEY is not configured`.
- Database connection issues are returned as readable errors (e.g., `Database connection failed`). The app attempts two common local connection strings before failing.
- If a truck or required column is not found, the app returns errors such as `Truck name column not found` or `Required columns not found`.
- If ORS fails to resolve locations, the app falls back to local `LovesLocations` database search.

---

## Troubleshooting & Notes

- Ensure `ODBC Driver 17 for SQL Server` is installed and a suitable SQL Server instance is reachable.
- The app tries `DESKTOP-7K48LCE` as server and then `localhost` as a fallback. Edit `get_db_connection()` if you need to connect to a different host or use SQL auth.
- Make sure `LovesLocations` contains useful address data to enable finding fuel stops along routes.
- If OpenRouteService rate limits your calls or returns unexpected responses, inspect the returned `error` in the function responses.

---

## Prerequisites

Before running the application, ensure the following requirements are met:

1. Install all required Python packages from the `requirements.txt` file:

   ```bash
   pip install -r requirements.txt
   ```

2. Obtain an OpenRouteService API key from OpenRouteService and add it to your environment variables.

3. Configure a valid database connection by providing a `DATABASE_URL`.

4. Create a `.env` file in the project root directory and add the following values:

   ```env
   API_KEY=your_openrouteservice_api_key
   DATABASE_URL=your_database_connection_url
   ```

5. Ensure SQL Server is accessible and the required tables (`Truck_Detail [Id, Name, Fuel_Average_L]`, `LovesLocations`, and `Trip_Records [Id, Starting_point, Destination_point, Total_Distance, Truck_Name, Truck_Average_L, Required_Time, Date]`) exist in the database.

---

## Run the Application

After completing the prerequisites, start the Streamlit application using:

```bash
streamlit run Distance-Fuel_calculator.py
```

The application will:

* Connect to the configured database.
* Use the OpenRouteService API for geocoding and route calculations.
* Calculate distance, fuel requirements, and recommended fueling stops.
* Store trip records in the database.
* Generate an interactive route map in the `generated_maps/` directory.


