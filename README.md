# README

#

# This repository contains a Jupyter notebook and several data files to build and solve a fleet‐optimization model for transporting commodities by sea. The model enforces commodity‐vessel compliance and covers trade demand between ports. Below is an overview of the notebook’s contents, the purpose of each data file, and how everything ties together.

#

# ---

#

# ## Overview

#

# 1. **Data Loading & Preparation**

# - Import raw data from Excel and CSV files.

# - Clean and reformat columns into Python‐friendly structures (lists, sets, dictionaries).

# - Build indexing structures (port lists, commodity lists, compliance sets, distance lookup, cleaning‐time lookup, etc.).

#

# 2. **Vessel Catalog & Technical Specifications**

# - Define dataclasses (`FuelCurve`, `VesselTechSpec`, `Vessel`) to hold each vessel’s capacity, fuel‐consumption curves (ballast vs. laden), and daily operating cost.

# - Generate a catalog of available vessel types (e.g., “tanker”, “bulker”, “combi”) and assign each a unique ID.

#

# 3. **Model Construction (Gurobi)**

# - Create binary decision variables for:

# - Purchasing/hiring each vessel.

# - Sailing each vessel between ports in ballast or laden (cargo) mode.

# - Loading/unloading triggers at ports.

# - At‐sea cleaning switches (`cb`) and in‐port cleaning switches (`cP`), with continuous “time‐at‐sea” variables for cleaning.

# - Whether each demand leg (origin → destination for a given commodity) is served.

# - Build the objective:

# - Maximize total revenue (from served trade legs) minus total cost, where cost includes:

# - Fuel consumption (based on distance, speed, and fuel‐cost-per‐ton).

# - Vessel operating expenses (daily opex when steaming or in port).

# - Port fees (per‐visit).

# - Handling costs (time in port, linking to daily opex).

# - Cleaning costs (sea vs. port cleaning).

# - Purchase/hire cost (CapEx).

# - Penalty for any unserved trade leg.

# - Add constraints for:

# - Flow conservation (departure from and return to home port).

# - Commodity‐compliance (a vessel only transports allowed commodities).

# - One cargo type per arc (when laden).

# - No two consecutive ballast legs without visiting a port.

# - Load/unload trigger logic (linking arc usage to port loading/unloading).

# - Cleaning logic (both at sea and in port) to enforce switching between commodities.

# - Time‐and‐voyage‐length limits (max stops, max days at sea + port).

# - Implement a lazy‐constraint callback to eliminate subtours (i.e., disconnected cycles that do not include the home port).

#

# 4. **Scenario Wrapper (`run_scenario`)**

# - Wrap the “build‐and‐solve” step into a single function.

# - After solving:

# - Extract scenario‐level KPIs (objective value, number of vessels bought, number of ballast legs, number of loaded legs).

# - Build a per‐vessel summary DataFrame including CO₂ metrics (CII, EEOI), total days, etc.

# - Generate interactive Plotly maps of active ballast vs. laden legs.

# - Print a human‐readable summary of each vessel’s operations (route, cargo movements, cleaning events, cost breakdown).

#

# ---

#

# ## File Descriptions

#

# 1. **Final\_export\_import\_data.xlsx**

# - Contains the master trade data:

# - Columns include `origin_port_id`, `dest_port_id`, `k` (HS6 commodity code), `rev_usd` (revenue per voyage), `leg_id`, plus geographic columns `lat_x, lon_x` (origin coordinates) and `lat_y, lon_y` (destination coordinates).

# - Used to build:

# - `legs` list (all feasible origin–destination–commodity combinations).

# - `coords` dictionary (port\_id → (latitude, longitude)).

#

# 2. **distance\_matrix.xlsx**

# - A square matrix of distances (nautical miles) between all ports.

# - First column is `port_id`, and subsequent columns are also port IDs (as strings).

# - Converted into a Python dictionary `d[(i, j)] = distance_nm` for quick lookup of any (origin, destination) pair.

#

# 3. **freight\_rate\_model.xlsx**

# - Contains freight‐rate schedules by commodity.

# - Columns:

# - `HS6` (commodity code),

# - `Distance_nm` (reference distance at which the “FR High” rate applies),

# - `FR low` (rate if voyage distance ≤ 1000 nm),

# - `FR High` (rate if voyage distance ≥ reference distance).

# - Used in the helper function `make_arc_revenue(...)` to compute expected revenue for each `(i, j, k)` arc, scaling by vessel capacity and interpolating between `FR low` and `FR High` based on the actual distance `d[(i,j)]`.

#

# 4. **exports\_by\_port.csv**

# - Two columns:

# - `port_id` (integer),

# - `k` (a string‐encoded list of HS6 codes, e.g. `"[100199, 251710, ...]"`).

# - Parsed to build `export_compat[port_id] = set_of_HS6_codes`.

# - We remove any occurrence of commodity code `151190` per business rules.

#

# 5. **imports\_by\_port.csv**

# - Same format as `exports_by_port.csv` but for imports.

# - Parsed to build `import_compat[port_id] = set_of_HS6_codes`.

# - Also strips out `151190`.

#

# 6. **asymmetric\_cleaning\_time\_matrix.xlsx**

# - A square matrix whose row/column headers are HS6 codes.

# - Entry `(k1, k2)` is the cleaning time (in days) required to switch from commodity `k1` to `k2`.

# - Converted into a dictionary:

# `cleaning_time[(int(k1), int(k2))] = time_in_days`

# - Used for both at‐sea cleaning (`cb` variables + `tSea`) and in‐port cleaning (`cP` and `resP` variables).

#

# 7. **Fuel\_usage\_vessels.csv**

# - Contains per‐vessel‐type fuel consumption rates (tonnes/day) at design speed, for ballast vs. laden conditions.

# - Combined with the vessel catalog to populate:

# - `fuel_b_tpd[vessel_id]  = fuel consumption in ballast (t/day)`

# - `fuel_l_tpd[vessel_id]  = fuel consumption when laden (t/day)`

# - `speed_ball[vessel_id]  = speed_ball_knots * 24  (nm/day)`

# - `speed_lad[vessel_id]   = speed_laden_knots * 24 (nm/day)`

#

# ---

#

# ## Notebook Structure

#

# Below is a high‐level outline of the notebook’s code cells and logical flow:

#

# 1. **Imports & Global Configuration**

# - `import pandas as pd`, `import gurobipy as gp`, `from collections import defaultdict, deque`, plus Plotly/SeaRoute for visualization.

# - Any global settings (e.g., Gurobi `options`, random seed, etc.).

#

# 2. **Data Loading & Initial Setup**

# - **Excel/CSV Reads**:

# \`\`\`

# data = pd.read\_excel('Final\_export\_import\_data.xlsx')

# N = pd.read\_excel('distance\_matrix.xlsx')\['port\_id'].to\_list()

# fuel\_usage = pd.read\_csv('Fuel\_usage\_vessels.csv')

# K\_export\_port = pd.read\_csv('exports\_by\_port.csv')

# K\_import\_port = pd.read\_csv('imports\_by\_port.csv')

# cleaning\_time\_matrix = pd.read\_excel("asymmetric\_cleaning\_time\_matrix.xlsx")

# freight\_rate\_df = pd.read\_excel('freight\_rate\_model.xlsx')

# \`\`\`

# - **Define Constants**:

# - `V = ['combi', 'tanker', 'bulker']`

# - `fuel_cost = 550   # $/tonne (default)`

# - Commodity list `K = [100199, 250100, ..., 271000]`

#

# 3. **Commodity‐Vessel Compliance**

# - Build `compliance_dict` mapping each vessel type string (`'combi'`, `'tanker'`, `'bulker'`) to a Python `set` of HS6 codes it is allowed to carry.

#

# 4. **Distance Matrix → Dictionary**

# - Convert `distance_matrix.xlsx` into `d[(i, j)] = float(distance_nm)` by iterating over `Distance_matrix.itertuples()`.

# - Key step for fast lookup in model constraints and objective.

#

# 5. **Export/Import Compatibility**

# - Clean the `'k'` column in `K_export_port` and `K_import_port` by stripping brackets/newlines and converting each entry to a `list[int]`.

# - Remove HS6 code `151190` from all lists.

# - Build two dictionaries:

# `export_compat = {port_id: set_of_HS6_codes}`

# `import_compat = {port_id: set_of_HS6_codes}`

#

# 6. **Cleaning‐Time Dictionary**

# - Read `asymmetric_cleaning_time_matrix.xlsx`.

# - Build `cleaning_time[(k1, k2)] = float(time_days)` for every pair of HS6 codes found in the matrix.

#

# 7. **Loading & Unloading Rates**

# - Hard‐coded dictionaries:

# `load_tpd = {100199: 22572, ..., 270900: 63216, 271000: 63216}`

# `unload_tpd = {100199: 17000, ..., 270900: 51840, 271000: 51840}`

# - These values (tonnes/day) represent port operations for dry‐bulk vs. liquid cargo.

#

# 8. **Coordinates Dictionary**

# - Using `data.itertuples()`, build `coords[port_id] = (latitude, longitude)` for all origin and destination ports.

# - Used for plotting routes on a map via Plotly + SeaRoute.

#

# 9. **Vessel Catalog & Tech Spec**

# - Define dataclasses:

# - `FuelCurve(speed_kn: float, fuel_tpd: float)`

# - `VesselTechSpec(capacity_t: int, ballast: FuelCurve, laden: FuelCurve, opex_usd_day: float)`

# - `Vessel(vessel_id: str, vtype: str, price_usd: float, tech: VesselTechSpec)`

# - Provide a dictionary `base_specs` for each ship type (capacity, price, opex, speed & fuel in ballast vs. laden).

# - Use `make_vessel_catalog(base_specs, n_each, prefix_map, seed)` to generate `fleet = { vessel_id: Vessel(...) }`.

#

# 10. **Helper Function: `make_arc_revenue`**

# - Given:

# - `distances[(i, j)]`,

# - `vessel_capacity` (capacity\_t),

# - `freight_rate_df` (HS6 → \[Distance\_nm, FR low, FR High]),

# - `trade_data` (DataFrame with columns `origin_port_id`, `dest_port_id`, `k`).

# - Build a dictionary `rev[(i, j, k)] = revenue_usd`, using piecewise/linear interpolation between `FR low` and `FR High` based on `distances[(i, j)]`, then multiplying by `vessel_capacity`.

#

# 11. **Subtour Elimination Callback (`SubtourEliminationCallback2`)**

# - A Gurobi callback that triggers at `MIPSOL`, detects any disconnected cycle (component) that does not include `home_port`, then adds a lazy constraint to prohibit that subtour.

# - Uses BFS/graph‐connectivity logic over the binary solution `x[v,i,j]`.

#

# 12. **Plotting Functions (`plot_routes_searoute_plotly`, `plot_active_legs_plotly2`)**

# - Visualize each vessel’s path on a world map:

# - Use `SeaRoute` to get a geo‐encoded shipping route between `(lat1, lon1)` and `(lat2, lon2)`.

# - Draw ballast legs (dashed red) vs. loaded legs (solid blue).

# - Mark port visits by HS6 code or port ID.

# - Highlight `home_port` with a black star.

#

# 13. **Model Constructor & Solver (`build_and_solve_fleet_model`)**

# - **Inputs**:

# - `home_port`, `demand_df`, `vessel_catalog`, `distances`, `fuel_cost`, `O_compliance_dict`, `port_fee`,

# - `load_tpd`, `unload_tpd`, `cleaning_time`, `cleaning_cost_pd`, `unserved_penalty`, plus solver parameters (`max_stops`, `max_voyage_days`, `timelimit`, `mipgap`).

# - **Steps**:

# 1. Build all sets (`V`, `Leg`, `P_use`, `arcsCARGO[v]`, `arcsBALLAST[v]`, `cb_index`, `cP_index`).

# 2. Instantiate Gurobi variables (`buy`, `x`, `xb`, `xl`, `cb`, `cP`, `tSea`, `yL`, `yU`, `serve`, `resP`).

# 3. Assemble objective components (revenue, fuel/opex/port/handling/cleaning/CapEx/penalty).

# 4. Add all constraints (linking `x` → `xb`/`xl`, single‐cargo‐per‐arc, flow conservation, no‐consecutive‐ballast, port triggers, cleaning logic, time/voyage limits).

# 5. Enable lazy constraints for subtour elimination.

# 6. Call `m.optimize(SubtourEliminationCallback2(...))`.

# - **Outputs**:

# \`\`\`

# return (

# m,

# V,

# vessel\_catalog,

# leg\_data,

# x, xl, xb, cb, cP, yL, yU, serve, buy,

# fuel\_cost\_expr, opex\_cost\_expr, port\_cost\_expr,

# handling\_opex\_expr, cleaning\_cost\_expr,

# capex\_expr, penalty\_cost\_expr, total\_cost,

# cii\_num, cii\_den, eeoi\_num, eeoi\_den,

# vessel\_days

# )

# \`\`\`

# These end‐user objects are used to compute KPIs and post‐process routes.

#

# 14. **Scenario Runner (`run_scenario`)**

# - Calls `build_and_solve_fleet_model(...)` with scenario‐specific parameters (home port, fuel price, time limit, MIP gap).

# - After solving:

# - **Scenario KPIs**: #legs, objective, #vessels bought, #ballast legs, #loaded legs.

# - **Vessel‐level summary**: calls `vessel_summary_df(...)` to get a Pandas DataFrame with metrics (CII, EEOI, vessel days, cargo ton‐nm, etc.).

# - **Plotting**: calls `plot_active_legs_plotly2(...)` and `pretty_print_fleet_solution(...)` to display routes and solution details.

# - Returns `(scenario_kpi, fleet_df)`.

#

# ---

#

# ## How to Run

#

# 1. **Install Dependencies**

# - Python ≥ 3.8

# - `pandas`, `gurobipy`, `plotly`, `searoute-py` (or equivalent), `numpy`

# - Ensure you have a valid Gurobi license and `gurobipy` installed.

#

# 2. **Place All Data Files in the Notebook Directory**

# - `Final_export_import_data.xlsx`

# - `distance_matrix.xlsx`

# - `freight_rate_model.xlsx`

# - `exports_by_port.csv`

# - `imports_by_port.csv`

# - `asymmetric_cleaning_time_matrix.xlsx`

# - `Fuel_usage_vessels.csv`

#

# 3. **Open the Notebook**

# - The notebook is organized into sections that mirror this README.

# - Run cells sequentially from top to bottom; each cell is commented to explain its purpose.

#

# 4. **Adjust Scenario Settings**

# - In the final cell, call:

# \`\`\`

# scenario\_kpi, fleet\_df = run\_scenario(

# demand\_df = data,        # or a filtered subset of 'data'

# scenario\_id = "BaseCase",

# compliance   = compliance\_dict,

# home\_port    = 0,

# fuel\_price   = 550,      # can override default 1000

# timelimit    = 3600,     # in seconds

# mipgap       = 0.05      # e.g. 5% optimality gap

# )

# \`\`\`

# - Inspect `scenario_kpi` (summary dict) and `fleet_df` (detailed vessel performance).

#

# 5. **Interpreting Outputs**

# - **`scenario_kpi`**:

# - `scenario`: scenario ID

# - `n_legs`: total number of demand legs in `demand_df`

# - `objective`: model’s objective value (revenue − cost)

# - `vessels_bought`: how many vessels were “bought” (binary `buy[v] = 1`)

# - `ballast_legs`, `loaded_legs`: number of arcs used in each mode

# - **`fleet_df`**: Contains one row per vessel with columns such as:

# - Vessel ID, type, CII, EEOI, total days (sea + port), total distance, total cargo ton‐nm, and utilization statistics.

#

# 6. **Visuals & Console Logs**

# - After solution, the notebook will:

# - Open interactive Plotly figures of each vessel’s route (ballast vs. loaded).

# - Print a detailed textual breakdown of each vessel’s operations, including cost components and cleaning events.

# - Display the vessel summary DataFrame in‐notebook.

#

# ---

#

# ## Key Points

#

# - **Commodity Compliance** is enforced via `compliance_dict[vessel_id]`, ensuring each vessel only carries allowed HS6 codes (and cannot pick up cargo whose code isn’t permissible).

# - **Distance‐Based Revenue** is calculated by linear‐interpolation between “FR low” (for ≤1000 nm) and “FR High” (for ≥reference distance) rates from `freight_rate_model.xlsx`.

# - **Cleaning Logic**:

# - If a vessel switches from commodity `k1` to `k2` in open sea (between two ballast arcs), at‐sea cleaning time `tSea[v,i,j,k1,k2]` is accounted for.

# - If a vessel switches in port, residual port cleaning time `resP[v,p,k1,k2]` is used (can only clean in port if it has arrived carrying `k1` and departs with `k2`).

# - Both at‐sea and in‐port cleaning costs are modelled as `cleaning_cost_pd * (time_spent)`.

# - **Subtour Elimination**: A callback ensures each vessel’s route is a single connected tour that starts and ends at `home_port` (no disconnected cycles).

# - **Penalties**: If a demand leg cannot be served (no available vessel or non‐compliant), the model pays a user‐specified penalty per unserved leg.

#

# ---

#

# ## File Summary

#

# - **Notebook**:

# - Step‐by‐step code to:

# 1. Load & clean all data files.

# 2. Build indexing structures: `N`, `V`, `K`, `compliance_dict`, `d`, `export_compat`, `import_compat`, `cleaning_time`, `load_tpd`, `unload_tpd`, `coords`.

# 3. Define dataclasses and helper functions (`FuelCurve`, `VesselTechSpec`, `Vessel`, `make_vessel_catalog`, `make_arc_revenue`, plotting functions, subtour callback).

# 4. Construct and solve the Gurobi model (`build_and_solve_fleet_model`).

# 5. Wrap everything into `run_scenario` for easy parameter sweeps and KPI extraction.

#

# - **Data Files**:

# - `Final_export_import_data.xlsx` – Master trade data + port coordinates

# - `distance_matrix.xlsx` – Compressed distance lookup among ports

# - `freight_rate_model.xlsx` – Freight‐rate interpolation data per HS6 code

# - `exports_by_port.csv` / `imports_by_port.csv` – Per‐port export/import allowed commodity codes

# - `asymmetric_cleaning_time_matrix.xlsx` – Cleaning‐time requirements (HS6 → HS6)

# - `Fuel_usage_vessels.csv` – Fuel consumption rates (ballast vs. laden) per vessel type

#

# ---

#

# ## Contact / Next Steps

#

# - To extend or modify the model, adjust:

# - **Data sources** (e.g., add new ports, new HS6 codes, updated distances).

# - **Vessel catalog** (e.g., change capacities, add new vessel types).

# - **Cost parameters** (fuel cost, port fees, opex, cleaning cost).

# - **Constraints** (e.g., journey‐length limits, additional operational restrictions).

# - For larger scenarios, consider:

# - Tightening `mipgap` for a more accurate solution.

# - Increasing `timelimit` to allow Gurobi to find better solutions.

# - Splitting into multiple sub‐scenarios (e.g., by region or commodity group).

#

# This README should help you understand how each piece of data and code fits into the overall commodity‐compliant fleet optimization framework. Good luck, and happy modeling!
