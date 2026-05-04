# Planning Intelligence — Knowledge Base

## Overview
Full-stack intelligence layer over Israeli planning & construction data.
Collects data from 10+ government/external sources → PostgreSQL+PostGIS → API.
Goal: answer "Is this location suitable for an energy facility?" in seconds.

---

## Core Entities & Data Sources

### 1. Plans (XPLAN) — Central Entity
- **Source**: iplan.gov.il ArcGIS REST API
- **Count**: ~35,824 plans
- **Key fields**: plan_number, plan_name, committee, status, dates, area_dunam, housing_units, main_land_use, geometry (EPSG:2039), score (0-100), score_breakdown (JSONB)
- **Related tables**: plan_documents (PDFs), plan_events (timeline), plan_changes (audit trail)
- **Statuses**: recommendation → pre_deposit → deposit → objections → approved → rejected
- **Ownership**: state / private / mixed (inferred from text)

### 2. Committee KPIs
- **Source**: Calculated from plans table (Materialized View)
- **Count**: ~136 committees
- **Metrics**: total_plans, approval_rate_pct, avg_days per stage, total_housing_units, avg_area_dunam

### 3. Demographics (Settlements)
- **Source**: CBS (Central Bureau of Statistics) via data.gov.il
- **Count**: ~1,185 settlements (2022 census + 2023 data)
- **Fields**: population (2023), pop_density, pop_growth_pct, age_median, age_young_pct (0-19), age_elderly_pct (65+), med_wage_annual_ils, academic_cert_pct, homeowner_pct, renter_pct, socioeconomic_cluster (1-10)
- **Link**: Each plan enriched with nearest settlement demo data

### 4. Real Estate Prices
- **Source**: nadlan.gov.il (Ministry of Justice)
- **Count**: ~400 settlements, 3-year rolling averages
- **Fields**: avg_price_3/4/5room_ils, trend_pct (3 years), market_yield_pct, luxury_index

### 5. Grid Infrastructure
- **Source**: TAMA_1, ArcGIS MAVAT
- **Components**:
  - `grid_transmission_lines`: voltage_kv (400/161/66/22kV), line_type, status, geometry
  - `grid_substations`: station_type (substation/power plant/pumped storage), geometry
  - `grid_gas_lines`: geometry
  - `electricity_licenses`: license_type (generation/storage/transmission), technology (solar/wind/storage/gas), capacity_mw, status
- **Function**: `grid_proximity(lon, lat)` → returns nearest 3 lines, 3 substations, 2 gas lines with distance in km

### 6. Energy Knowledge Base
- **Source**: Documents from Electricity Authority, Noga, Ministry of Energy
- **Count**: 42 documents → 19,096 chunks with embeddings
- **Categories**: energy_market_trade_tariffs (9), energy_storage_regulation (8), market_reports (6), facility_requirements (5), planning_statutory (5), noga_grid_planning (4), data_centers (4), iec_grid_connection (1)
- **Embedding**: 4096-dim (dictalm2.0-instruct:f16), cosine similarity + full-text fallback

### 7. Committee Protocols
- **Source**: complot.co.il + municipal websites
- **Count**: 10 municipalities → thousands PDFs → 19,096 chunks with embeddings
- **Feature**: `plan_number_ref` — chunks mentioning specific plans, linking protocols to plans

### 8. Site Intelligence
- **Facility types**: datacenter | storage | solar | wind
- **`site_synergy_score` (0-100)** weighted components:
  - Grid line proximity: 25
  - Substation proximity: 20
  - Compatible land use: 15
  - Anchor customers nearby: 15
  - Water access (cooling): 10
  - Urban infrastructure: 10
  - Security/safety: 5
- **`site_opposition_risk`**: scans protocol chunks within 15km for opposition keywords (התנגד, מחאה, קרינה, ערר, ביטול)
- **Anchor locations**: 95+ locations (banks, government, hi-tech, telecom, defense, industry, hospitals, universities, power plants, desalination, ports)

### 9. Solar & Climate
- **Source**: PVGIS (EU/JRC) + Open-Meteo
- **Coverage**: ~112 grid points (0.25° × 0.30°) over Israel
- **Fields**: ghi_kwh_m2_year, dni_kwh_m2_year, avg_temp_c, max_temp_summer_c

### 10. Land Zones (RMI)
- **Source**: Israel Land Authority → MAVAT
- **Types**: state_land, agricultural, nature_reserve, urban

### 11. Facility Requirements
- Per facility type (datacenter/storage/solar/wind) × land type (rami/local_authority/private)
- Categories: statutory, land_rights, infrastructure, environmental, permits

---

## Ontology — Entity Relationships

| Source | → | Target | Relationship |
|--------|---|--------|-------------|
| Plan | → | Committee | explicit (committee_code) |
| Plan | → | Demographics | spatial join (setl_code) |
| Plan | → | Prices | spatial join (setl_code) |
| Plan | → | Infrastructure | spatial proximity |
| Plan | → | Protocol | text match (plan_number_ref) |
| Plan | → | Solar/Climate | spatial nearest-neighbor |
| Energy Doc | → | Category | explicit (category) |
| Protocol chunk | → | Plan | text extraction (plan_number_ref) |
| Location | → | All layers | ST_DWithin / ST_NN spatial query |

---

## Data Confidence Tiers

| Tier | Description |
|------|-------------|
| **1** | Official government API, raw, no processing — plan numbers, dates, geometry, grid lines, land zones |
| **2** | Government data with our normalization — status_normalized, land_use_normalized, committee KPIs |
| **3** | Our computation — score, site_synergy_score, opposition_risk, accessibility_score |
| **4** | Reliable external with latency — CBS demographics (2022), nadlan prices (3yr avg), PVGIS solar |
| **5** | Partial coverage — anchor_locations, electricity_licenses, protocols (only 10/136 committees), 42 energy docs |

---

## Known Gaps

| Area | What's Missing |
|------|----------------|
| Protocols | Only 10 of 136 committees |
| Electricity demand | No actual consumption data |
| Energy licenses | Partial coverage |
| Transit stops | Not yet (TBD) |
| Arnona rates | Not yet (TBD) |
| Historic plans | XPLAN returns active/recent only |
| Rental prices | Sale prices only |

---

## Data Counts

| Domain | Table | Records |
|--------|-------|---------|
| Plans | plans | 35,824 |
| Committees | committee_kpis | 136 |
| Demographics | settlement_demographics | 1,185 |
| Infrastructure | grid_* | 127 |
| Energy docs | energy_documents | 42 |
| Protocol chunks | protocol_chunks | 19,096 |
| Anchor locations | anchor_locations | 95+ |

---

## Gas Infrastructure — Additional Sources (Lior's Research Request)

### Key Entity: INGL (נתג"ז / Netivei HaGaz)
- **Company**: Israel Natural Gas Lines — government corporation
- **Responsibility**: Plan, build, and operate the entire natural gas transmission system
- **System includes**: onshore pipelines, offshore pipelines, reception facilities from suppliers, metering stations
- **Website**: https://www.ingl.co.il/
- **Credit rating**: ilAAA (institutional investor grade)

### Available Public Data Sources
1. **INGL website** (ingl.co.il) — company data, pipeline projects, expansion plans
2. **data.gov.il** — Israeli government open data (GIS layers for gas infrastructure)
3. **MAVAT (מפת"ח)** — Government mapping center, may have updated gas line layers (already partially used via TAMA_1)
4. **Ministry of Energy** — Drilling licenses, transmission licenses, supply permits
5. **data-israeldata.opendata.arcgis.com** — Israeli ArcGIS open data portal (supports GeoJSON, KML, WMS, WFS)

### Current Data Gaps
- `grid_gas_lines` in the project comes from MAVAT TAMA_1 — may not be fully up-to-date
- No pipeline diameter/capacity data
- No connection point data (where customers connect)
- No gas pressure data (high/medium/low pressure lines)

---

## Electric Grid — IEC & Noga (Lior's Research Request)

### Key Entities

#### 1. IEC — Israel Electric Corporation (חברת החשמל לישראל)
- **Role**: State-owned utility — builds, maintains, and operates ALL generation, transmission, and distribution
- **Scope**: Largest electricity supplier in Israel + Palestinian territories
- **100 years** of operational history
- **Second largest procurement org** in Israel (5,000+ active suppliers worldwide)
- **Website**: https://www.iec.co.il/
- **IEC Global**: https://iec-global.com/

#### 2. Noga — Israel Independent System Operator (נגה - ניהול מערכת החשמל)
- **Role**: Independent government corporation — system operation, planning, development, trading
- **Established**: Mid-2021 (electricity reform)
- **Responsibilities**: Real-time grid operation, remote supervision of generation/transmission/substation, grid resilience, economic optimization, renewable integration
- **Website**: https://www.noga-iso.co.il/
- **Key function**: Manages the grid with high renewable penetration (solar variability, cloud events)

### Grid Infrastructure (IEC System)

#### Voltage Levels
- **400 kV** — National backbone transmission
- **161 kV** — Primary transmission network
- **66 kV / 22 kV** — Sub-transmission and distribution
- Numerous GIS-based substations (modern insulation technology)

#### Power Generation
- IEC responsible for generation fleet (coal → natural gas transition underway)
- High short-circuit levels (strong grid)
- Energy island (limited interconnection — Egypt, Jordan gas pipelines but electricity grid mostly isolated)

### Available Data Sources
1. **IEC website** (iec.co.il) — corporate info, customer services
2. **Noga website** (noga-iso.co.il) — system operation data, planning documents
3. **MAVAT TAMA_1** — Already in the project:
   - `grid_transmission_lines` (400/161/66/22kV, line types, status)
   - `grid_substations` (substation, power plant, pumped storage)
   - `electricity_licenses` (generation, storage, transmission, with technology + capacity_mw)
4. **Ministry of Energy** — Regulates licensing of generation, transmission, distribution
5. **Rashut HaHashmal** (רשות החשמל) — Economic regulation, tariffs, market design
6. **Electricity Maps** (app.electricitymaps.com) — API for real-time grid mix and carbon intensity data globally, including Israel
7. **Open Infrastructure Map** (openinframap.org) — OSM-based world electricity infrastructure map

### Current Data Gaps
- No real-time grid load data (Noga/IEC operational data is proprietary)
- No distribution-level (22kV and below) network data
- No power plant output data (generation mix by fuel type)
- No demand-side / consumption data at facility level
- No interconnection capacity data (Egypt/Jordan links)

### Grid Development Plans (2030)
- **ILS 17 billion (~$4.5B)** plan from Ministry of Energy
- **Double** the number of 400 kV transmission lines
- **Expand** 161 kV lines by 30%
- **Increase** substations and switching stations by ~50%
- **Support** renewable energy integration (solar, storage)
- Source: IEA policy database

### Industry Context (2026)
- Israel's gas production hitting record highs in 2026
- Major expansions: Leviathan field, Tamar field
- New pipeline upgrades to increase exports to Egypt and Jordan
- $2.36B Leviathan expansion, 15-year $35B supply deal with Egypt
- Offshore Technology: GlobalData tracks Israel's gas sector in MENA report

---

## Water Infrastructure — Israel Water Authority (Lior's Research Request)

### Key Entities

#### 1. Israel Water Authority (רשות המים)
- **Established**: 2007
- **Role**: Central regulator — consolidates all water and sewage authority in Israel
- **Responsibility**: Water sector regulation, planning, tariffs, desalination policy, wastewater treatment
- **Website**: https://www.gov.il/he/departments/water_authority
- **English info**: https://www.gov.il/he/pages/water-authority-data-english

#### 2. Mekorot (מקורות) — National Water Company
- **Role**: State-owned — national water supply company, operates the bulk water system
- **Responsibilities**: Desalination absorption, water distribution, national water carrier, developing Israel's water map
- **Scope**: Everything Mekorot does serves Israeli households, agriculture, and industry
- **Website**: https://www.mekorot-int.com/

### Israel's Water Sector — Key Stats

#### Water Sources
- **~50% of supply** comes from unconventional sources (desalination + reclaimed wastewater)
- Conventional sources (Sea of Galilee, aquifers) no longer meet demand
- Long drought 1998-2002 triggered massive desalination investment

#### Desalination — Current + Planned
- **2022 capacity**: ~596 MCM/year (million cubic meters per year)
- **2050 target**: ~1,700 MCM/year (nearly 3x increase)
- Major plants: Hadera, Ashkelon, Sorek (among world's largest)
- Mekorot absorbs desalinated water into the national system

#### Wastewater
- Advanced treated effluent infrastructure development
- High percentage of wastewater reuse for agriculture (world-leading)

#### Water Tech Leadership
- Israel is a global water-tech powerhouse
- Full-stack ecosystem: desalination, wastewater reuse, smart networks, digital monitoring, precision irrigation
- Exports water solutions worldwide

### Available Data Sources
1. **Water Authority gov.il page** — Sector model, reform info, desalination overview
2. **data.gov.il** — Israeli open data portal (search "water" for datasets on consumption, infrastructure, quality)
3. **data-israeldata.opendata.arcgis.com** — Israeli ArcGIS open data (may have water infrastructure GIS layers)
4. **GovMap API** (api.govmap.gov.il) — Government mapping portal, may have water pipe layers
5. **Mekorot website** — Corporate info, project updates

### Relevance to Planning Intelligence Project
- **Water access for cooling** is a component in `site_synergy_score` (10pts max for facility suitability)
- Desalination plants are anchor customers (anchor_locations)
- Water infrastructure proximity matters for:
  - **Data centers** (cooling water requirements)
  - **Energy storage** (pumped storage needs water)
  - **Power plants** (thermal cooling)
  - **Industrial facilities**

### Current Data Gaps in the Project
- No dedicated water_infrastructure table (pipes, reservoirs, desal plants as GIS layers)
- No water consumption data per facility or region
- No water quality / pressure data
- No sewage infrastructure data
- No desalination plant capacity/status GIS layer
- Water_rights/water_licenses — no data
- arnona_rates: TBD
- transit_stops: TBD

### Grid Development Plans (2030) — already indexed above under Electric Grid

## 🧠 Anton's Core Framework — Investment & Arbitrage Logic (Lior's Edge)

### The Fundamental Insight
Everyone thinks: **"Land → Project"**
The data says: **"Grid connection = the bottleneck"**
Your arbitrage: Identify **connection + timing + right use** before the market prices it in.

---

### 🔥 5 Real Arbitrage Sources (Based on Reports + Data)

#### 1. ⚡ Grid Arbitrage (Strongest)
**Everyone**: "Land = Project"
**Reports say**: Grid connection is the REAL bottleneck (Energy Ministry, Nofar, everyone hints at this)
**Your arbitrage**:
- Where is there **unused capacity**?
- Where is there a **nearby substation NOT saturated**?
- Where are there **approved connections NOT utilized**?
- **Strategy**: Go opposite the market — everyone runs to area X, you find where capacity is actually available

#### 2. 🔋 Storage Timing Arbitrage
**Everyone**: Adding storage because "it's needed"
**Reports say**: Storage is exploding NOW (Doral etc.)
**Your arbitrage**:
- Areas with lots of solar WITHOUT storage yet
- Areas with **grid congestion** → storage = solution
- **Edge**: Enter early where storage isn't yet priced correctly

#### 3. 🏗️ Land Mispricing (RMI / Municipalities)
**Everyone**: Looks at land price
**You see**:
- Connection potential
- Zoning change potential
- Municipal interest (wants tax revenue)
**Arbitrage**: Find land that's:
- Cheap because "nobody knows what to do with it"
- BUT: close to grid, borderline zoning, municipality wants development → gold

#### 4. 🏢 Data Center Arbitrage (Very New)
**Market not pricing yet**: The connection between:
- Electricity
- Land
- Fiber
- Storage
**Insight**: Data center = simply "huge reliable electricity consumer"
**Arbitrage**: Find:
- Industrial land
- With strong grid
- Near fiber
- Add storage possible
→ Before Mega Or / Gav-Yam arrive

#### 5. 🧠 Information Arbitrage (Most Important)
**Everyone**: Reads reports
**You**: Connect reports + GIS + RMI + Planning + Grid
**Real example**:
- Company reports: "Project in area X" + "Connection difficulty"
- You: Identify WHERE X is + find WHERE connection IS available nearby
→ Go there before everyone else

---

### 💡 Your Practical Advantage

**You're building**: Not a "land database" → but a **Decision Engine**

| Instead of: | You say: |
|-------------|----------|
| "Good land here" | "Connection in 18 months, zoning almost ready, municipality wants this, no competition" |

### 🧠 Product Output Format
For EVERY site:
- 🔥 **Grid Score** (most important)
- 🟡 **Statutory Complexity**
- 🔵 **Infra Readiness**
- 🟢 **Commercial Potential**
- 🔴 **Risk**

Then: **GO / WAIT / NO-GO**

---

### 🚨 Where Everyone Is Wrong (Your Edge)

| Mistake | What they do | What you do |
|---------|-------------|-------------|
| ❌ #1 | Looking for **land** | Looking for **capacity** |
| ❌ #2 | Running after **tenders** | Running after **timing** |
| ❌ #3 | Don't understand **grid** | Grid is your **superpower** |

### 🎯 One Sentence Summary
> Your arbitrage = early identification of **connection + timing + right use** (energy/storage/data center) — before the market prices it in.

### Electricity Reform — Market Liberalization (2025-2026)

#### 1. The Electricity Revolution (מהפכת החשמל) — Now Live
- **Biggest electricity reform** since the cellular reform in Israel
- Every consumer can **choose their electricity supplier** (not just IEC)
- Expected savings: **5-20% off the electricity tariff** per household
- Hundreds to thousands of shekels saved per household annually

#### 2. Consumer Mobility (ניוד צרכנים) — How It Works
- System name: **PLA** (פל"א — Platform for Automatic Mobility)
- Process:
  1. Consumer contacts chosen private supplier
  2. Supplier submits mobility request via PLA platform (Noga)
  3. Noga checks data against IEC systems
  4. Digital power of attorney (ייפוי כוח דיגיטלי) sent to consumer
  5. Approved within **7 business days**
  6. Takes effect on **1st of following month**
- **Smart meter required** for standard mobility
- **Basic meter**: also possible since July 2024 (Minister of Energy decision)
  - Requires submitting self-reading photo within set window
- Smart meter installation: via IEC (call 103), one-time fee: ₪234.37 (per Electricity Authority tariff table 3-5.4)

#### 3. List of Private Suppliers
- Available from Noga website as PDF (מספקי חשמל פרטיים לצרכנים ביתיים)
- Consumer can return to IEC by:
  - Asking supplier to terminate contract (takes effect next month + 1)
  - Calling IEC 103 directly to terminate (takes effect next month)

#### 4. Discounts & Benefits After Mobility
- **50% discount** on first 400 kWh (for eligible households) — still applies after switching
- Noga receives eligibility lists from relevant institutions monthly
- Consumer can submit eligibility letter to supplier if not on list

#### 5. Ancillary Services (שירותים נלווים) — Noga's New Initiative
- Noga developing a **computerized platform** for ancillary services trading
- Goal: support **30% renewable energy target**
- Challenges: renewable variability (solar cloud events, ramp rates)
- Platform will enable trading of: frequency regulation, reserves, voltage control

### Power System Facts & Figures (2025)

#### Demand Records
- **August 2025**: All-time record peak — **16,970 MW** (3 consecutive days)
- Previous record (Aug 2023): 15,694 MW
- During peak: **4,038 MW** from renewables (significant contribution to grid resilience)

#### Generation Mix
- IEC owns ~75% of total generation capacity
- Transitioning from coal to natural gas:
  - Rutenberg (Ashkelon): 2,200 MW — 4 units converting
  - Orot Rabin (Hadera): 1,100 MW — 2 units converting
  - New gas units replacing old coal units (Mizramim 70 & 80 at Orot Rabin)
- Private producers: Dorad, Dalia, OPC (Rotem, Hadera), Adelteq (Ramot Negev, Ashdod), IPM, Alon, MRC

#### System Overview
- **Founded**: 1923 by Pinchas Rutenberg (103 years of operation)
- **Status**: Government-owned (99.85% state)
- **Employees**: ~9,782 permanent + 2,894 temporary
- **Integrated utility**: Generation, transmission, distribution (monopoly)
- **Headquarters**: Haifa (IEC Tower)
- **CEO**: Ofer Bloch
- **Chairman**: Yiftah Ron-Tal

### Regulatory & Licensing

#### Electricity Authority (רשות החשמל)
- Sets tariffs, market rules, licensing conditions
- Publishes tariff books (ספר התעריפים) — tables updated periodically
- Determines mobility rules and timelines
- Ancillary services regulation

#### Ministry of Energy
- Strategic oversight: 30% renewables target, coal phase-out
- 2030 transmission plan: ILS 17 billion (~$4.5B)
- Renewable integration roadmap

#### Noga ISO (since 2021)
- Independent system operator
- Operations, planning, trading
- Market operator: wholesale electricity market (MCP pricing)
- New MCP (Market Clearing Price) methodology published for public comment (deadline: Feb 2026)
- System planning: transmission, generation adequacy, renewables integration

### Connection Requirements for Facilities

When siting a facility near IEC infrastructure (per project checklist):
- **IEC expert opinion** required (חוות דעת חברת חשמל)
- **Transformer requirement assessment** (דרישת השנאה / חדר טרפו)
- **Existing infrastructure mapping** (מיקום תשתיות קיימות)
- **Route coordination** (תיאום תוואים)
- Connection point approval process
- High-voltage vs. medium-voltage connection: depends on facility load (MW)

### Data Sources Available
1. IEC website (iec.co.il) — corporate info, customer service (site is JS-heavy, limited scraping)
2. Noga ISO (noga-iso.co.il) — system operation, market data, renewable integration, reform updates
3. Electricity Authority (gov.il/energy) — tariff books, licensing, market regulation
4. Ministry of Energy — strategic planning, 2030 grid plan, fuel switching
5. Wikipedia — historical context, corporate structure
6. MAVAT TAMA_1 — already in project (transmission lines, substations, licenses)
7. GlobalData — IEC T&D project database (premium)

### What's Already in Your Project
- `grid_transmission_lines` — 400/161/66/22kV lines with geometry
- `grid_substations` — substations, power plants, pumped storage
- `electricity_licenses` — generation/storage/transmission with technology + MW
- `grid_proximity()` — nearest lines and substations

---

## Substations in Israel — Full Map (Lior's Research)

### Grid Voltage Levels (already indexed)
- **400 kV** — Backbone transmission (newest, highest capacity)
- **161 kV** — Main transmission (historically the backbone)
- **66 kV** — Regional distribution
- **22 kV** — Local distribution (to neighborhoods, industrial zones)

### Major 400/161 kV Substations (by Region)

| Substation | Location | Voltage Levels | Notes |
|------------|----------|:--------------:|-------|
| **Haifa** | חיפה | 400/161 kV | IEC HQ area, serves Haifa Bay industry |
| **Tel Aviv** (Tzrifin/Yavne corridor) | מרכז | 400/161 kV | Major load center |
| **Ashkelon** | אשקלון | 400/161 kV | Near Rutenberg power plant |
| **Hadera** (Orot Rabin) | חדרה | 400/161 kV | Near coal-to-gas conversion plant |
| **Jerusalem** | ירושלים | 161 kV | Capital load center (growing) |
| **Be'er Sheva** | באר שבע | 161 kV | Negev hub for solar parks |
| **Dimona** | דימונה | 161 kV | Industrial + research center |
| **Eilat** | אילת | 161 kV | Southernmost, isolated grid segment |
| **Caesarea** | קיסריה | 400 kV | New 400kV backbone station |
| **Rosh HaAyin** | ראש העין | 400 kV | New, near planned Kesem station |
| **Karmiel** | כרמיאל | 161 kV | Northern development town |
| **Afula** | עפולה | 161 kV | Jezreel Valley hub |
| **Kfar Saba** | כפר סבא | 161 kV | Sharon region |
| **Rishon LeZion** | ראשון לציון | 161 kV | Dense urban area |
| **Ashdod** | אשדוד | 161 kV | Port + industrial zone |
| **Ramla/Lod** | רמלה/לוד | 161 kV | Near Gezer power plant |

### 400 kV Backbone Stations (The "New Ring")
The 400 kV system is the newest and most strategic:
1. **Caesarea** (400/161 kV) — main backbone node
2. **Rosh HaAyin** (400 kV) — new, serves center-east
3. **Haifa** (400/161 kV) — northern anchor
4. **Ashkelon** (400/161 kV) — southern anchor (near Rutenberg)
5. **Tel Aviv corridor** (400/161 kV) — highest demand area
6. **Hadera** (400/161 kV) — near Orot Rabin

### Substation Functions
| Type | Purpose | Typical Voltage |
|------|---------|:---------------:|
| **Step-up** | Connect power plants to transmission grid | Plant → 161/400 kV |
| **Step-down** | Distribute to regions | 400/161 kV → 66/22 kV |
| **Switching station** | Route power without transformation | 400 kV or 161 kV |
| **Distribution substation** | Feed neighborhoods/industry | 66/22 kV → 0.4 kV |
| **Mobile substation** | Temporary capacity for construction | Various |

### How Substation Capacity Affects Projects
- Each substation has a **finite capacity** (MVA rating of transformers)
- Saturation means: **no new connections possible** without upgrade
- Upgrading a substation = **3-7 years** (planning, approvals, construction)
- New substation = **5-10 years**
- **Key insight**: A substation at 80%+ capacity is a RED FLAG
- A substation at 40-60% capacity with good road access = GOLD

### Grid Bottleneck Detection (Practical)
When evaluating a site:
1. Which substation serves this area? (closest based on proximity)
2. What's its estimated load? (no public data, but inferred from recent approvals)
3. Is there an approved upgrade plan? (TAMA_1 2030 plan = ILS 17B investment)
4. Are there recently approved connections that hint at remaining capacity?

### Current Grid Status
- **Northern region**: Relatively more capacity (less industrial load)
- **Central region**: Saturated (Tel Aviv metro, highest demand density)
- **Jerusalem corridor**: Growing (new neighborhoods, light rail)
- **Southern region**: High solar injection → congestion (needs storage + 400kV expansion)
- **Eilat**: Isolated, limited capacity (connects via 161 kV line through Arava)

### Already in Your Project
- `grid_substations` — all known substations as GIS point features
- `grid_transmission_lines` — all voltage levels with geometry
- `grid_proximity()` — function to find nearest substations to any point
- Energy docs (42) — contains Noga grid planning documents

### Data Gap
- **No substation load data** (MVA utilization %) — proprietary/IEC internal
- No transformer-level connectivity data
- No feeder-level maps below 22 kV
- No queue of pending connection requests
- **Recommendation**: Infer from MAVAT TAMA_1 data + recent approvals + news

---

## 🏛️ Planning & Building Law — Israel (חוק התכנון והבנייה)

### Core Framework
- **Enacted**: August 12, 1965 (תשכ"ה-1965)
- **Amendments**: 137 to date
- **Ministry**: Ministry of Interior (משרד הפנים)
- **Full text**: Wikisource (2MB+) + Knesset database

### Key Concepts

#### 1. Planning Hierarchy
| Level | Body | Plans |
|-------|------|-------|
| **National** | מועצה ארצית לתכנון ולבנייה | TAMA (תמ"א) — National Outline Plans |
| **District** | ועדה מחוזית לתכנון ולבנייה | District Outline Plans |
| **Local** | ועדה מקומית לתכנון ולבנייה | Local Plans / תב"ע |

#### 2. Permit Requirements (היתר בנייה)
- Any building/use requires a permit from the local committee
- **Criminal penalties**: up to 2 years imprisonment for building without permit
- **Administrative enforcement**: "Kaminitz Law" (תיקון 116, 2017)
  - Allows demolition orders, administrative fines, immediate stop-work orders
  - 41% reduction in violations after implementation
  - 1,264 administrative fine demands in 2019 alone
- **Amendment 136** (2022, "Electricity Law"): enables connection to electricity/water/phone even without building permit for structures built before 2018 in designated areas (primarily Arab settlements)

#### 3. Brownfields (קרקעות חומות) — Definition & Regulation
**Not explicitly defined in the Planning Law** — the term "brownfield" is not a statutory term in Israel.

**However, the following laws and regulations apply to contaminated/reused industrial land:**

| Category | Regulation | Key Points |
|----------|-----------|------------|
| **Soil contamination** | חוק שמירת הניקיון (Cleanliness Law) | Remediation before change of use |
| **Industrial waste** | חוק פסולת תעשייתית | Waste treatment requirements |
| **Hazardous materials** | חוק החומרים המסוכנים | Storage/transport permits |
| **Water contamination** | חוק המים | Groundwater protection |
| **Environmental impact** | חוק התיישנות | Liability for past contamination |

**Practical implications for brownfield redevelopment:**
1. Must perform **soil survey** (סקר קרקע) before any construction
2. **Remediation plan** required if contamination found
3. **Environmental impact assessment** may be required
4. **Planning committee** evaluates based on:
   - Current land use (industrial)
   - Proposed use (energy storage / data center / residential)
   - Proximity to sensitive receptors (residential, water sources)
5. **Liability**: Current owner may be liable for past contamination from previous industrial use

**Who evaluates**: המשרד להגנת הסביבה (Ministry of Environmental Protection) + local committee
**Typical timeline**: addition of 6-18 months for remediation before construction

#### 4. Storage Facilities (מתקני אגירה) — Legal Status

**Energy storage regulation** is **new** — no specific chapter in the Planning Law.

**Relevant frameworks that apply:**

| Framework | What it covers |
|-----------|----------------|
| **Energy Sector Law** (חוק משק החשמל) | Storage licenses, grid connection |
| **Electricity Authority regulations** | Market participation, tariff structure |
| **Planning Law** (general clauses) | Land use, building permits for storage structures |
| **Fire safety regulations** | Battery storage fire safety (critical) |
| **Environmental regulations** | Battery disposal, cooling systems, noise |
| **TAMA 1** (תמ"א 1) | Siting near transmission infrastructure |
| **TAMA 10** (תמ"א 10/ב') | Siting near hazardous facilities |
| **INGL coordination** (since July 2021) | Mandatory if near gas lines |
| **Grid connection rules** | Noga / IEC connection requirements |

**Key facts for storage facilities:**
- **No specific "storage" land use designation** currently exists in the Planning Law
- Storage is typically permitted under **"energy facility"** or **"industrial"** designations
- **Local committees** have wide discretion on a case-by-case basis
- **Fire safety** is the #1 concern for battery storage (lithium-ion fire risk)
- **Recommended approach**: verify land use in existing תב"ע + consult local committee before purchase

#### 5. Data Centers — Legal Status
- No specific law for data centers
- Typically: **"Industrial" or "Employment" (תעסוקה)** land use
- Grid connection is the main bottleneck (IEC expert opinion required)
- Fiber connectivity: private arrangements with Bezeq / HOT / cell carriers
- **No special planning treatment** — treated as industrial building with high power demand

#### 6. Key Planning Documents That Apply

| Document | Relevance |
|----------|-----------|
| **TAMA 1** (תמ"א 1) | Transmission lines, substations — mandatory coordination |
| **TAMA 10/ב'** | Hazardous facilities, gas pipelines |
| **TAMA 14** (תמ"א 14) | Waste disposal facilities |
| **TAMA 35** (תמ"א 35) | Comprehensive national plan (updated) |
| **District plans** | Vary by district (North, Haifa, Center, Tel Aviv, Jerusalem, South) |
| **Local תב"ע** | Specific land use designations per lot |

#### 7. Practical Navigation Flowchart (Storage / Data Center Project)

```
1. Identify land → 
2. Check existing תב"ע (planning-intelligence database has 35K plans) →
3. Verify land use designation →
   a. "Energy" or "Industrial" or "Employment" → proceed
   b. "Agricultural" or "Open space" → zoning change needed (6-18 months)
4. Check proximity to:
   - Grid (substation with capacity) → grid_proximity() function
   - Gas lines → INGL coordination (mandatory if <500m)
   - Residential areas → opposition risk
   - Environmental sensitivity → EIA may be required
5. Apply for permit:
   - Full set of plans (architectural, engineering)
   - Fire safety appendix (critical for storage)
   - Environmental appendix (if required)
   - Traffic appendix
   - IEC expert opinion (electricity)
6. Timeline estimate: 12-24 months for standard project
```

#### 8. Where the Gaps Are (Lior's Edge)

| What exists | What's missing |
|-------------|----------------|
| Planning Law general framework | Specific storage provisions |
| Environmental regulations | Battery safety standards (evolving) |
| IEC grid connection rules | Standardized storage connection process |
| Local committee discretion | No "fast track" for energy facilities |
| Fire safety regulations | Lithium-ion battery specific codes (international standard NFPA 855) |
| 35K plans in database | Not all plans are digitized electronically |

**Strategic insight**: The **lack of specific regulation** for storage and data centers works in YOUR favor if you understand the existing frameworks better than competitors. Most developers wait for clear rules — you can move faster.

---

## ✈️ Airports & Aviation — Impact on Energy Planning (Seveso / Land Use)

### Overview
Energy facilities (storage, generation) near airports face **height restrictions, safety buffer zones, and obstacle limitation surfaces** under ICAO regulations, Israel Airports Authority (IAA) rules, and Israeli planning laws.

### International Airports in Israel

| Airport | Location | IATA/ICAO | Runways | Key Restriction Zone |
|---------|----------|-----------|---------|---------------------|
| **Ben Gurion (נתב"ג)** | Lod, Central | TLV/LLBG | 3 (4,062m, 3,112m, 2,772m) | **Largest** — extends ~15-18km radius |
| **Ramon** (רמון) | Be'er Ora, South | ETM/LLER | 1 (3,600m) | Opened 2019, desert area |
| **Haifa** (חיפה) | Haifa | HFA/LLHA | 1 | Coastal, near industry |

### Active Military Airbases (All affect land use near them)

| Airbase | Location | Notes |
|---------|----------|-------|
| **Hatzerim** | Be'er Sheva | Near Negev solar fields |
| **Hatzor** | Hatzor/Ashdod | Central, near industry |
| **Nevatim** | Nevatim, South | Major IAF base |
| **Ovda** | Uvda, South | Near Eilat/Ramon |
| **Ramat David** | North | Major base |
| **Ramon** | Mitzpe Ramon | Negev |
| **Tel Nof** | Rehovot | Central |
| **Palmachim** | Coast near Rishon | Spaceport + airbase |
| **Sdot Micha** | Jerusalem District | |
| **Ein Shemer** | Haifa District | |

### Closed Airports (Relevant for Redevelopment)

| Airport | Location | Closed | Notes |
|---------|----------|--------|-------|
| **Sde Dov** (שדה דב) | Tel Aviv | 2019 | Redevelopment = one of Israel's largest residential projects |
| **Eilat** (old) | Eilat | 2019 | Replaced by Ramon |
| **Atarot** (עטרות) | Jerusalem | 2001 | Reopening discussed — **critical**: Keystone power station proposed in Atarot Industrial Zone |
| **Kiryat Shmona** | North | Inactive | |

### Key Regulatory Frameworks

#### 1. ICAO Annex 14 — Obstacle Limitation Surfaces
- **All countries signatory to Chicago Convention must implement** incl. Israel
- Defines restricted surfaces around airports:
  - **Outer Horizontal Surface**: radius up to 15km from airport, height limit = 150m (300ft) above airport elevation
  - **Conical Surface**: sloping outwards from 150m to 350m (8-15km from airport)
  - **Approach Surface**: extending 15km along landing path
  - **Transitional Surface**: along runway length
- **Energy facilities**: solar panels can cause glare for pilots, wind turbines have height limits, batteries need fire safety near airports
- **Communication towers/masts**: height permits require IAA approval

#### 2. TAMA (תמ"א) Regulations Relevant to Airports

| TAMA | Subject | Relevance to Energy |
|------|---------|---------------------|
| **TAMA 1** | Infrastructure | Transmission lines near airport paths |
| **TAMA 7** (תמ"א 7) | **Civil Aviation** | Directly regulates airport protection zones, approach surfaces, obstacle heights |
| **TAMA 10/ב'** | Hazardous facilities | Siting near air transport infrastructure |
| **TAMA 35** | National comprehensive | General land use designations |
| **TAMA 40** | Water/sewage/AEMA | |

**TAMA 7 key provisions** (תמ"א 7 — תכנ�ת מטאר ארצית לתעופה):
- Defines **protection zones** around airports (אזורי מגן לשדות תעופה)
- Limits building heights within approach paths
- Prohibits certain hazardous facilities within defined zones
- Requires coordination with **Civil Aviation Authority (CAA)** for any structure >certain height within airport vicinity
- No specific mention of energy storage but falls under "hazardous" facilities

#### 3. Israel Airports Authority (IAA) — Land Use Near Airports
- IAA reviews any **building permit application** within defined airport influence zones
- **Coordination required** for: height above 45m within 5km of airport, above 90m within 10km, antennae and masts
- Solar installation glare analysis may be required
- **Battery storage** fire safety must be reviewed for proximity to aviation fuel storage
- Ben Gurion Airport has the **largest protection zone** (~18km radius effective)

#### 4. Israel Defense Forces (IDF/IAF) — Military Airbase Infringement
- Military airbases have **secrecy restrictions** — their protection zones are less publicly documented
- Any tall structure near an IAF base = **requires security clearance** (משרד הביטחון)
- The **Southern Negev** (near Ramon, Ovda, Nevatim) has extensive military airspace restrictions
- **Practical impact**: Solar farms near Hatzerim/Nevatim = possible IAF objection due to:
  - Flight path interference
  - Glare from solar panels
  - Communication interference

### What This Means for Energy Storage / Solar Projects

| Scenario | Risk Level | What to Verify |
|----------|:-----------:|----------------|
| Site within 5km of Ben Gurion | 🔴 HIGH | Height limits, IAA coordination, approach surface |
| Site within 5km of military airbase | 🔴 HIGH | Height limits + MOD security clearance |
| Site near closed airport (redevelopment) | 🟢 LOW-MED | Check if TAMA 7 still applies |
| Site in Negev near Ramon/Ovda | 🟡 MED | IAF flight zones, solar glare permits |
| Site in central Israel, >10km from any airport | 🟢 LOW | Standard building permit process |
| Solar farm near any airfield | 🟡 MED | Glare analysis mandatory |

### Practical Navigation Checklist
```
For ANY energy/storage project near an airport or airbase:
1. Determine distance from nearest airport/airbase
2. If < 15km from Ben Gurion → mandatory IAA coordination
3. If < 10km from military base → mandatory MOD/IAF clearance
4. Check height limit (usually 45m within 5km → specific calc needed)
5. For solar → glare analysis
6. For battery storage → fire safety review + aviation fuel proximity
7. Timeline addition: 3-6 months for airport/IAF coordination
```

### Already in Your Project
- GIS can calculate distances to known airports (point features)
- TAMA 1 (transmission) and TAMA 10/B' (hazardous) already indexed
- Planning committee data (136 committees) — some will have airport-related conditions

### Data Gaps
- **No digitized TAMA 7 protection zone polygons** in public GIS
- Military airbase exact obstacle limitation surfaces = classified
- No detailed IAA coordination procedures (available only per application)
- **Recommendation**: Create GIS buffer layers for known airports (15km = outer horizontal surface for Ben Gurion, 10km for military bases)

### Strategic Insight
> Airports and IAF bases create **"land use shadow"** — areas where development is restricted. This depresses land prices. If you identify zones where:
> - The restriction doesn't actually affect your project type (e.g. battery storage in a low-obstacle configuration)
> - Or the restriction is about to be lifted (closed airport redevelopment)
> → You buy **mispriced land** (Arbitrage #3 — Land Mispricing).

---

## 🗺️ Strategic Sites for Energy Storage — Cross-Referenced Analysis

### Methodology: Layering Committee + Grid + Land Use + Airport

Layer 1 (highest weight): **Substation capacity + proximity** (Grid Score)
Layer 2: **Committee approval speed** (Statutory Complexity)
Layer 3: **Land use compatibility** (Already industrial/energy vs needs rezoning)
Layer 4: **Airport/IAF restrictions** (Height, safety zones)
Layer 5: **Competition** (How many projects already in the area)

---

### Tier 1: 🔥 HIGHEST PRIORITY — Immediate Arbitrage

| Site | Reasoning | Grid | Statutory | Airport | Competition |
|------|-----------|:----:|:---------:|:-------:|:----------:|
| **Be'er Sheva / Negev North** | 161kV substation, solar congestion → storage needed, industrial land cheap, Hatzerim concern manageable | 🟢 HIGH | 🟢 MED | 🟡 IAF | 🟢 LOW |
| **Dimona area** | 161kV, existing solar, industry, far from airports | 🟢 HIGH | 🟢 MED | 🟢 LOW | 🟢 LOW |
| **Rosh HaAyin** | New 400kV! Planned Kesem station (780MW). Land still cheap. Zoning industrial. | 🟢 V.HIGH | 🟢 MED | 🟢 LOW | 🟡 MED |
| **Karmiel / North** | 161kV, spare capacity, north needs storage, municipal interest | 🟢 HIGH | 🟢 MED | 🟢 LOW | 🟢 LOW |

### Tier 2: 🟡 STRONG — Good, but more complexity

| Site | Grid | Statutory | Airport | Competition |
|------|:----:|:---------:|:-------:|:----------:|
| **Jerusalem corridor** (Motza/Atarot) | 🟡 MED | 🔴 HIGH | 🔴 HIGH | 🟡 MED |
| **Caesarea / Hadera** (400kV backbone, but competitive) | 🟢 HIGH | 🟢 MED | 🟢 LOW | 🔴 HIGH |
| **Ashdod** (port + industry + Hatzor airbase) | 🟡 MED | 🟢 MED | 🟡 Hatzor | 🔴 HIGH |
| **Ashkelon** (400/161kV, Rutenberg conversion → capacity) | 🟡 MED | 🟢 LOW | 🟢 LOW | 🔴 HIGH |

### Tier 3: 🟢 NICHE — Special situations

| Site | Grid | Statutory | Airport | Competition |
|------|:----:|:---------:|:-------:|:----------:|
| **Eilat** (isolated grid, very low competition) | 🟡 MED | 🟢 MED | 🟡 Ramon | 🟢 V.LOW |
| **Kiryat Gat** (Intel power station planned = grid upgrade) | 🟡 MED(future) | 🟢 MED | 🟢 LOW | 🟢 LOW |
| **Closed airports** (Sde Dov, Eilat old, Atarot) | 🟢 HIGH | 🔴 HIGH | 🟢 LOW | 🟢 LOW |
| **Kfar Sava / Sharon** (saturated center but peak shaving) | 🟡 MED | 🟢 MED | 🟢 LOW | 🟡 MED |

---

### Cross-Reference: Committees by Area

| Location | Likely Committee | Complexity |
|----------|-----------------|:----------:|
| Be'er Sheva | ועדה מקומית באר שבע | MED — used to energy |
| Dimona | ועדה מקומית דימונה | LOW — few projects, fast |
| Rosh HaAyin | ועדה מקומית ראש העין | MED — growing city |
| Karmiel | ועדה מקומית כרמיאל | MED — development town |
| Jerusalem | ועדה מחוזית ירושלים | 🔴 HIGH + Atarot |
| Caesarea | מוא�ורית חוף הכרמל | MED |
| Ashdod | ועדה מקומית אשדוד | MED |
| Eilat | ועדה מקומית אילת | LOW — pragmatic |
| Kiryat Gat | ועדה מקומית קרית גת | LOW — Intel helps |
| Sde Dov redev | ועדה מקומית תל אביב | 🔴 HIGH — urban |

---

### Strategic Recommendations

#### Short-term (Next 6m) — GO:
1. **Be'er Sheva / Dimona corridor** — Search RMI tenders here
2. **Rosh HaAyin** — Before Kesem 780MW starts (land will 2-3x)
3. **Karmiel** — Low competition, spare grid, municipality wants growth

#### Medium-term (6-18m) — WATCH:
4. **Jerusalem corridor** — Wait for Atarot decision + Keystone clarity
5. **Eilat** — Monitor desal + storage tenders
6. **Kiryat Gat** — Intel power station approval → adjacent land

#### Long-term bet — NICHE:
7. **Sde Dov redevelopment** — Storage as temporary use (5-10yr) on land in planning. Low land cost, premium location.

### How Your Data Validates This

| What you have | How to use |
|---------------|------------|
| 35,824 plans | Filter: plans near these substations with industrial/energy use |
| grid_substations | Calculate: which have lowest project density nearby |
| grid_proximity() | Run: for any RMI lot → nearest substation + capacity |
| electricity_licenses | Check: licenses approved near each substation |
| Protocols (10 comm�tees) | Read: objections filed for energy projects |
| 42 energy docs | Cross-ref: Noga planning for upgrade plans |

### Data Gaps
- [ ] Protocol data for ALL 136 committees (currently only 10)
- [ ] RMI tender feed (Cloudflare blocked)
- [ ] Substation load % (MVA util) — estimate from approvals
- [ ] Noga/PUA transformer upgrade schedule
- [ ] Detailed industrial land inventory by committee

---

## 🏗️ Committees — Full Requirements & Restrictions for Energy Storage (Lior's Training)

### The Core Insight
Committees have **no specific chapter** in the Planning Law for energy storage. This means:
- They apply **existing frameworks** (industrial/energy facility)
- They make **case-by-case decisions**
- The **variation between committees** is where arbitrage lives

### What Committees Actually Publish for Storage Facility Applications

Every application requires a **permits file (תיק היתר)** containing:

#### 1. Core Engineering Documents (Always Required)
| Document | Form | Purpose |
|----------|------|---------|
| **Architectural plans** | Drawings (.dwg/.pdf) | Layout, sections, elevations of storage units |
| **Survey (תשריט מדידה)** | Signed by licensed surveyor | Lot boundaries, existing infrastructure |
| **Floor area table** | Excel/std form | Land coverage, building footprint |
| **Zoning compliance** | Declaration | Adherence to existing תב"ע |
| **Letter of commitment (כתב התחייבות)** | Legal form | Applicant's commitment to conditions |
| **Land ownership proof** | Tabu / RMI confirmation | Property rights |

#### 2. Committee-Specific Appendices for Storage Facilities

| Appendix | Required by | When | Key Issues for Storage |
|----------|------------|------|-----------------------|
| **Fire safety (בטיחות אש)** | ALWAYS | Every storage app | Battery fire containment, cooling, access for fire trucks, water supply, NFPA 855 compliance |
| **IEC connection (חברת חשמל)** | ALWAYS | Every storage app | Transformer sizing, grid connection point, feeder capacity |
| **Traffic (תנועה)** | ALMOST ALWAYS | If access changes | Truck movement during construction, battery transport weight |
| **Accessibility (נגישות)** | ALWAYS | Every building permit | Access paths to control rooms, emergency exits |
| **Environmental (סביבה)** | LOCAL COMMITTEE | Near residential / water sources | Battery cooling emissions, noise from inverters, waste disposal |
| **Acoustic (אקוסטי)** | LOCAL COMMITTEE | If near residential (<300m) | Inverter hum, HVAC noise |
| **INGL coordination (נג"ז)** | MANDATORY | If within 500m of gas line | Safety buffer zones per ASME B31.8 |
| **Drainage (ניקוז)** | LOCAL COMMITTEE | On larger lots | Runoff management from impervious surfaces |
| **Parking (חניות)** | ALWAYS | Every app | Minimal for storage (low occupancy) but required |

#### 3. Storage-Specific Conditions Committees Impose

| Condition | Typical Requirement | Impact on Project |
|-----------|-------------------|-------------------|
| **Battery type restriction** | Only LFP (LiFePO4), no NMC near residential | Limits technology choice |
| **Setback from property line** | 5-15m (varies by committee) | Reduces usable land % |
| **Height limit** | Usually 8-12m (container stacking limit) | Volume constraint |
| **Fire wall requirement** | 2-hour fire rated wall between battery banks | Cost adder |
| **Sprinkler system** | NFPA 13 compliant, with foam suppression | Major cost adder |
| **Water retention pond** | For firefighting runoff containment | Land requirement |
| **Emergency access road** | 6m minimum width for fire trucks | Land + grading |
| **Security fence** | 2.5m+ with anti-climb | Security cost |
| **Noise limit** | 55 dBA at property line (typical) | Inverter/HVAC specs |
| **Lighting restriction** | No spillover to adjacent residential | Low-profile lighting |
| **Operational hours** | Only daytime for delivery/transport | Logistics constraint |
| **Connection to IEC** | Medium voltage (22kV) for <20MW, high voltage (161kV) for >20MW | Infrastructure cost |
| **Bond/guarantee** | Bank guarantee for dismantling at end of life | Financial commitment |
| **Decommissioning plan** | Required as part of permit | Long-term liability |

#### 4. How Committees Vary (This Is Your Edge)

| Dimension | Lenient Committee (GOOD) | Strict Committee (BAD) |
|-----------|------------------------|------------------------|
| **Processing speed** | 3-6 months | 12-24 months |
| **Setback requirement** | 5m from property line | 15-25m (Jerusalem, Tel Aviv) |
| **Height limit** | 12m (allows double-stack containers) | 6-8m (urban committees) |
| **Fire suppression** | NFPA 13 sprinklers only | Additional foam system required |
| **Environmental review** | Basic declaration | Full EIA (Environmental Impact Assessment) |
| **Opposition risk** | Low — few residents nearby | High — NIMBY |
| **Land cost** | Low (peripheral) | High (central) |
| **Grid capacity** | Likely available | Likely saturated |

#### 5. Committee Archetypes (Based on Data + Geography)

| Type | Examples | Speed | Restrictions | Best For |
|------|----------|:-----:|:------------:|----------|
| **⏩ Fast-track peripheral** | Dimona, Eilat, Arad, Kiryat Gat | 3-6m | LOW | Storage > solar |
| **🟡 Industrial zone specialist** | Ashdod, Haifa Bay, Mishor Rotem | 6-12m | MED | Storage + data |
| **🔴 Urban complex** | Tel Aviv, Jerusalem, Rishon | 12-24m | HIGH | Data center only |
| **🟢 Development town** | Karmiel, Afula, Kiryat Shmona | 4-8m | MED | Storage (lower cost) |
| **🟡 Mixed suburban** | Rosh HaAyin, Kfar Sava, Rehovot | 6-12m | MED | Storage near new 400kV |

#### 6. How to Read Committee Protocols for Storage Applications

Protocols (פרוטוקולים) published after committee meetings contain:
1. **Applicant details** — who submitted, project name
2. **Appendices submitted** — what documents were filed
3. **Committee positions** (עמדות הוועדה) — objections, conditions
4. **Objections from public** — NIMBY concerns
5. **Required supplementary documents** — what they're missing
6. **Final decision** — approve / approve with conditions / reject

**What to look for in protocols for YOUR decision:**
| Signal | What it means |
|--------|---------------|
| "דורשת השלמות" (requires supplements) | Time waster — 3-6m delay |
| "התנגדות תושבים" (resident objection) | Risk of lengthy hearings |
| "בכפוף לתנאי כיבוי אש" (subject to fire conditions) | Standard — fire appendix is key |
| "דורש תיאום נג"ז" (requires INGL coordination) | Gas line nearby — check proximity |
| "אושר" (approved) | Look at conditions — learn the pattern |
| "נדחה" (rejected) | Understand why — avoid same mistakes |

#### 7. Decision Template: When I Evaluate a Location

```
Location: [name]
Committe: [name]
Grid Score: 🔥 / 🟡 / 🟢 (based on substation proximity + capacity)
Statutory: 🔴 / 🟡 / 🟢 (committee type + land use compatibility)
Airports: 🔴 / 🟡 / 🟢 (distance to airport/airbase)

KEY QUESTIONS:
1. Is land zoned industrial/energy? → If not, +12-24m for rezoning
2. Distance to nearest substation with capacity? → grid_proximity()
3. Gas lines nearby? → If <500m, INGL coordination needed (+3-6m)
4. Residential proximity? → If <300m, acoustic + environmental + opposition risk
5. Airport within 15km? → Height restriction + IAA coordination
6. Any prior storage approvals in same committee? → Pattern to follow

PREDICTED OUTCOME: GO / WAIT / NO-GO
Timeline estimate: [X] months
Key risk: [fire / environmental / grid / NIMBY / airport]
Max attractive land price: [ILS/sqm] based on grid + infra + risk
```

#### 8. What's Already in Your Project

| Data | How It Helps |
|------|-------------|
| 35,824 plans | Filter: storage/energy/industrial plans by committee |
| 10 committee protocols | Read: pattern of conditions per committee |
| 42 energy documents | Regulatory framework for storage connection |
| grid_substations + grid_proximity() | Calculate grid availability for any location |
| electricity_licenses | See what's already been approved near a site |

#### 9. Learning Loop

Every time you ask me about a specific location, I will:
1. Apply this entire framework
2. Cross-reference all 8 layers (grid, statutory, airports, land use, committee type, environmental, gas, competition)
3. Give you: **GO / WAIT / NO-GO** with timeline + key risk
4. Ask if you want me to dig deeper on specific appendix requirements for that site

The more specific locations you test, the more I refine the committee patterns.


---

## 📋 Real Tender Analysis: Mishkal 2025 (מכרז משכ"ל 2025) — Technical Spec for Energy Systems

### Document Source
- **Tender ID**: אמ/24/2025
- **Publisher**: Mishkal (משכ"ל) — Israel's purchasing organization for local authorities
- **Scope**: PV solar + energy storage + OFF GRID + ON GRID systems for public buildings

### Key Organizational Requirements

**Required Professionals (Contractor Must Employ):**
| Role | License | Scope |
|------|---------|-------|
| Electrical Engineer | חשמלאי מהנדס, full-time | All electrical design, compliance with Electricity Law 1954 |
| Civil Engineer | רשום | Foundations, concrete, columns for canopies |
| Mechanical Engineer / Constructor | קונטרוקטור | Steel structures, trusses, load-bearing elements |

**Site Access:**
- 48-hour advance coordination with municipality for every entry
- Written approval required for surveys, lifts, construction, maintenance

**Rooftop Restrictions:**
- ABSOLUTE PROHIBITION on welding/cutting metal on waterproofed roofs
- Equipment transport: pneumatic-wheeled carts only
- All hoisting by crane with verified accessibility

### Solar Panel Technical Specs

| Parameter | Requirement |
|-----------|-------------|
| Minimum power | 615W peak |
| Tier | Tier 1 (Bloomberg) |
| Technology | Poly or Mono crystalline |
| Min efficiency | 20% standard, 16.75% flexible |
| Temperature coefficient | < -0.41%/°C |
| Tolerance | Positive only |
| Bifacial | At no extra cost |
| PID Free | Manufacturer cert required |
| Standards | IEC 61215, IEC 61730, Israel Standards Institute |

**Flexible panels (Apollo Power or equiv):**
- Weight ≤3 kg/m² panel, ≤4 kg/m² system
- Hail: IEC 61730, class E2, ≤3.4%
- Bend: IEC 61215, 1m diameter
- Fire rating: broof(t3)
- Spare parts: 5% of installed qty kept in stock

**Warranty:** Product 10yr, linear power 25yr (max 20% degradation)
**Insurance:** International (e.g. Power Guard), ≥20yr, covers manufacturer bankruptcy

### Inverter Requirements
| Parameter | Requirement |
|-----------|-------------|
| Brands | SolarEdge or equiv (extra cost) |
| Standards | TUV, CE, VDE 0126-1-1 |
| Phases | Three-phase |
| Min efficiency | 98% |
| Power factor | Adjustable to cos φ = 1 |
| Install height | 50-200 cm from surface |
| Location | Accessible, shaded, locked, theft-protected |
| Enclosure | IP65 or weatherproof |
| Warranty | Manufacturer, ≥12 years |

### Cable Specs
- **DC**: Double-insulated, self-extinguishing, UV-protected, TUV/VDE, min 6mm², continuous runs
- **AC**: XLPE copper (N2XY) or aluminum (NA2XY), per IS 1516
- **Conduits**: Per IS 61386
- **Loss budget**: 1% max each DC and AC (at 70°C)
- **Surge protection**: Class II on DC side
- **Switchgear**: ABB DC or equiv; 4-pole AC breakers

### Grid Connection (IEC)
- Contractor manages connection upgrade with IEC
- Client bears upgrade costs per Mishkal price list
- Inverters must be IEC-approved

### Fire Safety (Instruction 543) — Referenced
- Compliance with Ministry of Education fire regulations for educational buildings
- Generator integration: automatic PV disconnection on generator activation
- Full Instruction 543 text is Appendix A (pages 51-54, not fully extracted)
- **Storage-specific pages 26-29 and OFF/ON GRID protocols pages 30-38 also not yet extracted**

### Strategic Implications for Storage Siting
1. Any storage project must meet or exceed these requirements
2. Use this spec to baseline equipment + engineering costs
3. Grid connection is the rate-limiting step — contractor manages it
4. Instruction 543 fire safety is the key appendix — need full text
5. Municipal projects follow this pattern → storage projects in same committees face similar requirements






---

## ⚖️ Regulatory Framework: Israeli Electricity Sector

### Structure Overview

| Entity | Hebrew | Est. | Role |
|--------|--------|:----:|------|
| **Public Utility Authority (PUA)** | רשות החשמל | 1996 | Independent regulator: tariffs, licensing, standards |
| **Noga ISO** | נגה - מנהל המערכת | 2021 | Independent System Operator: grid management, dispatch, balancing |
| **IEC (Israel Electric Corp)** | חברת החשמל | 1923 | Monopoly utility: generation, transmission, distribution (75% gen capacity) |
| **Private Producers** | יצרנים פרטיים | ~2010 | ~25% of generation (natural gas, solar, wind) |
| **Ministry of Energy** | משרד האנרגיה | — | Government policy: energy independence, renewables targets, gas strategy |

### PUA — Public Utility Authority for Electricity (רשות החשמל)

**Legal Basis**: Electricity Market Law 1996 (חוק משק החשמל, התשנ"ו-1996)
**Replaced**: IEC's 70-year Rutenberg concession (expired 1996)

**Key Responsibilities for Energy Storage:**
1. **Licensing** — All electricity generation/storage facilities need a license or exemption
2. **Tariffs** — Sets feed-in tariffs, grid connection fees, storage remuneration rates
3. **Regulation** — Defines technical standards for grid connection, safety, metering
4. **Renewable energy quotas** — Implements govt targets (30% renewable by 2030)
5. **Consumer protection** — Service quality standards

**Relevant PUA Determinations for Storage:**
- Storage licensing framework (what permits are needed per MW)
- Grid connection fees for storage facilities
- Mandatory technical standards for inverter connections
- Smart meter requirements (mandatory for new installations)
- Tariff structure that incentivizes peak shaving

**Chairperson**: Dr. Assaf Eilat (since 2016)
**Key Department**: Engineering Dept — handles connection feasibility, smart grid, reliability

### Noga — Independent System Operator (נגה - מנהל המערכת)

**Legal Basis**: Amendment to Electricity Market Law, established 2018, operational 2021
**Role**: Separated from IEC to create an independent grid operator

**Key Functions for Storage:**
1. **Grid dispatch** — Operates the system, balances supply-demand in real-time
2. **Connection approvals** — Determines if a storage facility can connect to the grid (capacity, location, timing)
3. **System planning** — Publishes development plans (תוכנית פיתוח), identifies where storage is needed
4. **Reactive power** — Sets voltage control and reactive power requirements
5. **Ancillary services** — Defines storage participation in frequency regulation, reserve capacity
6. **Day-ahead scheduling** — Coordinates generation + storage dispatch

**Key Documents:**
- Development Plan (תוכנית הפיתוח) — multi-year grid expansion including storage targets
- Connection Procedure (תהליך החיבור לרשת) — for new storage + solar facilities
- Technical Requirements Document (דרישות טכניות לחיבור) — tells you everything needed
- Mobility Procedure (תהליך הניוד) — for switching electricity suppliers

**Storage Relevance**: Noga determines WHERE storage is needed most (grid bottlenecks, overloaded substations) — this is the most valuable regulatory input for your site selection

### IEC — Israel Electric Corporation (חברת החשמל)

**Status**: ~99.85% state-owned, largest employer in the sector (~13K employees)
**Infrastructure**: 50 power stations, 22.2 GW capacity, 76.9 TWh/year (2022)

**Relevant for Storage:**
1. **Grid connection (physical)** — IEC builds the physical connection from substation to facility
2. **Transformer capacity** — IEC controls substation transformer loading — they know utilization %
3. **Feeder availability** — IEC manages the medium-voltage and high-voltage feeders
4. **Metering** — New installations require smart meters (IEC installs)
5. **Connection timeline** — Standard connection: 3-12 months depending on complexity
6. **Cost estimation** — IEC provides cost estimates for grid reinforcement if needed

**Connection Process (For Storage):**
1. Submit request to IEC (with PUA license or exemption)
2. IEC feasibility study (2-4 weeks)
3. Cost estimate for connection (2-4 weeks)
4. Complete infrastructure works (3-12 months depending on distance to substation, transformer availability)
5. Commissioning and testing (1-2 months)

**Bottleneck**: Substation transformer upgrade = 3-7 years. If your site requires a new transformer, the timeline blows up.

### Authority Boundaries: Who Decides What for Storage

| Issue | Decided By | Key Consideration |
|-------|-----------|-------------------|
| Can I build a storage facility? | LOCAL COMMITTEE | Planning Law, land use, zoning |
| Can I connect to the grid? | NOGA + IEC | Grid capacity, substation load, feeder availability |
| How much will grid connection cost? | IEC | Distance to substation, required upgrades |
| What tariff do I get? | PUA | Tariff determination, storage remuneration |
| Is gas coordination needed? | INGL | Distance to gas pipeline, safety buffers |
| Fire safety requirements? | LOCAL COMMITTEE + Fire Authority | NFPA 855, Instruction 543 (PV) |
| Environmental approval needed? | LOCAL COMMITTEE + Ministry of Environment | EIA for projects >50MW |

### Planning Law (חוק התכנון והבנייה, תשכ"ה-1965) — Key Architecture

**Basis**: 137 amendments, created the three-tier planning system:

| Level | Body | Plans | Relevance to Storage |
|-------|------|-------|---------------------|
| **National** | מועצת התכנון העליונה | TAMA (תמ"א) | TAMA 10 (energy), TAMA 7 (aviation) |
| **District** | ועדה מחוזית | District plans | Designates energy zones, regional storage |
| **Local** | ועדה מקומית | Local plans, building permits | APPROVES YOUR STORAGE PROJECT |

**Key Articles for Storage:**
- **Article 1-6**: Definitions — "energy facility" category exists but "storage" not specifically defined
- **Article 62**: Building permit requirements — applies to all structures including battery containers
- **Article 63**: Permitted uses — storage falls under "energy facility" or "industrial" depending on size
- **Article 78**: Change of use — converting land to energy use
- **Article 145**: Enforcement — penalties for building without permit (up to 2 years imprisonment)
- **Article 151**: Conditional permit — allows storage as temporary use on land designated for future development

**TAMA 10 (תמ"א 10)**: National plan for energy infrastructure — specifically addresses:
- Power plants, substations, gas pipelines
- Does NOT explicitly address battery storage (yet)
- Does address pumped storage (hydro)
- Creates mechanisms for national-level approval of large energy facilities

**Strategic Implication**: Storage falls between the cracks of TAMA 10 and local plans. This means:
- Small storage (<16MW?): local committee approves (faster)
- Large storage (>50MW?): may require district or national involvement (slower)
- The threshold is not clearly defined — this is Lior's edge (argue for local committee jurisdiction)

### Key Laws and Regulations (Summary)

| Legislation | Year | Relevance to Storage |
|-------------|:----:|---------------------|
| Planning & Building Law | 1965 | Permitting, zoning, building permits |
| Electricity Market Law | 1996 | PUA authority, licensing, tariffs |
| Amendment (Noga) | 2018 | ISO establishment, grid management |
| Natural Gas Sector Law | 2002 | INGL coordination for gas proximity |
| Cleanliness Law | 1984 | Brownfield remediation on contaminated land |
| Hazardous Materials Law | 1993 | Battery handling, disposal (lithium) |
| Water Law | 1959 | Water proximity, drainage |

### What This Means for Site Selection

1. **PUA license ≠ building permit** — You need both. They're separate processes.
2. **Noga decides grid availability** — This is THE gating factor. If Noga says "no capacity at this substation," nothing else matters.
3. **IEC builds the connection** — Budget 3-12 months for connection works.
4. **Local committee decides the building permit** — If the land is zoned industrial/energy, they can approve without rezoning.
5. **Thresholds matter** — Small vs large storage goes through different approval paths.
6. **The gap is your edge** — No clear regulation for storage = you interpret the framework favorably.

### Data You Have vs. What You Need

| You Have | You Need | How to Get It |
|----------|----------|---------------|
| 35K+ plans | PUA licenses issued for storage | PUA website (blocked) |
| 136 committees | Noga grid capacity map | Noga development plan |
| 42 energy docs | IEC substation load data | IEC connection feasibility |
| grid_substations | INGL pipeline GIS data | INGL coordination request |
| 10 protocols | Instruction 543 (fire safety) | Fire Authority / tender doc |
| electricity_licenses | Storage tariff rates | PUA tariff book |
| settlements | Land ownership per lot | RMI (blocked) |


- Energy knowledge base — 42 docs (19K chunks) on regulation, tariffs, connection requirements
- Protocol data — 19K chunks with committee positions on energy issues

### Data Gaps
- No real-time grid load data (Noga operational data is proprietary)
- No distribution-level network data (below 22kV)
- No power plant output by fuel type
- No facility-level consumption data
- No detailed transformer/feeder connectivity
- No tariff history data (structured)
- No IEC GIS layers beyond what's in TAMA_1

---

## Power Plants in Israel — Full Map (Lior's Research)

### Coal Power Stations (IEC)

| Station | Location | Capacity (MW) | Status |
|---------|----------|---------------|--------|
| **Orot Rabin** | חדרה - Hadera | 2,590 | Converting to natural gas (since 2016) |
| **Rutenberg** | אשקלון - Ashkelon | 2,250 | Converting to gas (2025-2026, 4 units, ~2,200 MW) |

### Gas-Fired Combined Cycle (CCGT) — IEC Stations

| Station | Location | Capacity (MW) | Commissioned |
|---------|----------|---------------|:------------:|
| **Gezer** | רמלה - Ramla | 1,332 | 1998–2008 |
| **Hagit** | עין תות, כרמל - Highway 6 | 1,030 | 1996–2007 |
| **Eshkol** | אשדוד - Ashdod | (IEC, details vary) | |
| **Haifa** | חיפה - Haifa | (IEC) | |
| **Reading** | תל אביב - Tel Aviv | (IEC) | |
| **Alon Tavor** | (IEC) | | |

### Gas-Fired — Private Producers (IPPs)

| Station | Location | Capacity (MW) | Commissioned |
|---------|----------|---------------|:------------:|
| **Dorad** | דרום אשקלון - S. Ashkelon | 800 | 2013–2014 |
| **Dalia** | כפר מנחם - Kfar Menahem | 860 | 2015 |
| **Tzafit** | כפר מנחם - Kfar Menahem | 595 | 1991–2012 |
| **Mishor Rotem** | מישור רותם - Mishor Rotem | 440 | 2013 |
| **Ramat Hovav** | נאות חובב - Neot Hovav | 520 | 1989–1999 |
| **OPC Rotem** | (private) | (MW varies) | |
| **OPC Hadera** | חדרה | (MW varies) | |
| **Adelteq — Ramat Negev** | רמת נגב | (MW varies) | |
| **Adelteq — Ashdod** | אשדוד | (MW varies) | |
| **IPM — Be'er Tuvia** | באר טוביה | (MW varies) | |
| **Alon Energy Centers** | (various) | (MW varies) | |
| **MRC** | (private) | (MW varies) | |

### Planned / Under Construction Gas Stations

| Station | Location | Planned Capacity (MW) | Status |
|---------|----------|:---------------------:|--------|
| **Kesem** | ראש העין - Rosh HaAyin | 780 | Pre-construction (approved 2023) |
| **Shimshon** | בית שמש - Beit Shemesh | est. 830 | Approved (Ashtrom to build, Jul 2025) |

### Solar Power Plants

| Plant | Location | Capacity (MW) | Technology | Commissioned |
|-------|----------|:-------------:|------------|:------------:|
| **Ashalim Plot A** | נגב - Negev | 121 | CSP Parabolic Trough + 4.5h storage | 2019 |
| **Ashalim Plot B** | נגב - Negev | 121 | CSP Tower (50,600 heliostats, 260m) | 2019 |
| **Ashalim Plot C** | נגב - Negev | 30 | PV | 2018 |
| **Ketura Sun** | קיבוץ קטורה | 4.95 | PV | 2011 |
| **Neot Hovav** | נאות חובב | (various) | PV | |
| **Tze'elim** | צאלים | (various) | PV | |
| **EDF Renewables (Ashalim)** | נגב | ~40 | PV | 2019 (record low 8.68 agorot/kWh) |
| *All residential solar* | nationwide | 1,900+ (estimated) | Rooftop PV | growing |
| *Utility-scale solar* | nationwide | ~3,500+ (2025 est.) | PV + CSP | expanding |

### Pumped Storage Hydroelectricity

| Station | Location | Capacity (MW) | Status |
|---------|----------|:-------------:|--------|
| **Kokhav HaYarden** | בית שאן - Beit She'an | 344 | Operational |
| *Additional planned pumped storage* | (multiple sites) | est. 1,000+ | In planning / approval |

### Nuclear

| Facility | Location | Capacity (MW) | Notes |
|----------|----------|:-------------:|-------|
| **Shimon Peres Negev Center** | דימונה - Dimona | 0 (research) | Research reactor, no power generation |

### Total Estimated Generation Capacity (2025)
| Fuel Type | Capacity (MW) | % of Total |
|-----------|:-------------:|:----------:|
| Natural Gas (IEC + IPPs) | ~10,000 | ~62% |
| Coal (Orot Rabin + Rutenberg, pre-conversion) | ~4,840 | ~30% |
| Solar (all types) | ~4,500 | ~7% |
| Pumped Storage | ~344 | ~1% |
| **Total** | **~16,200** | **100%** |

### Notable Context
- **Peak demand record** (Aug 2025): 16,970 MW
- **Renewables during peak**: 4,038 MW (solar)
- **Coal phase-out**: complete by 2025-2026 (remaining units converting to gas)
- **30% renewable target**: by 2030 (currently ~12-15% of generation)
- **Natural gas**: 70% of electricity generation (2018 figure, higher now)
- **Annual consumption**: 57,149 GWh (2017), growing ~2-3% yearly
- **Installed capacity**: ~16.25 GW (2014 base, growing)
- **Grid interconnection**: Limited — Egypt gas pipeline + Jordan (no significant electricity interconnector)

### Data Sources
- Wikipedia (list of power stations, individual station pages)
- IEC corporate data
- Noga ISO operational reports
- Global Energy Observatory archives
- Times of Israel, Globes (news on new stations)
- MAVAT TAMA_1 (transmission layer — already in project's grid_substations)

### What's Already in Your Project
- `grid_substations` — includes power station locations as point features
- `electricity_licenses` — includes generation licenses with capacity_mw, technology, status
- Anchor customers — power plants are anchor_locations
- Energy documents — many IEC/Noga related PDFs in the 42 energy docs

---

## INGL — Natural Gas Regulatory Framework for Storage Facilities (Lior's Research)

### Core Legislation

#### 1. Natural Gas Sector Law — חוק משק הגז הטבעי, התשס"ב-2002
- Primary legislation governing the entire natural gas sector in Israel
- Grants INGL a **30-year license** (since 2004) for transmission system
- Authority: Minister of Energy
- Governs: licensing, safety, tariffs, transmission, distribution, storage

#### 2. INGL Transmission License (רישיון ההולכה)
- Granted by Minister of Energy in 2004, under the Natural Gas Sector Law
- Exclusive right to plan, build, and operate the national gas transmission system
- **~900 km** of underground high-pressure pipelines
- **54 pressure reduction & metering stations (PRMS)**
- **~100 branch valves** along the system
- Standards: International pipeline standards (ASME, ISO)

### Key Regulatory Requirements for Facilities Near Gas Lines

#### 1. Mandatory Coordination (תיאום תשתיות) — ALL activities require it
- **Every action near gas lines requires individual coordination with INGL**
- Since July 2021: All coordination through **National Infrastructure Coordination System** (מערכת תיאום תשתיות לאומית)
- Process: Submit plan through the system → INGL reviews → issues engineering approval or excavation permit
- Applies to: municipalities, government companies, public bodies, private entities

#### 2. Pre-Approval Documents Required
- **General guidelines** for detailed planning coordination (הנחיות כלליות לתכנון מפורט)
- **Technical guidelines** (הנחיות טכניות לתכנון מפורט)
- **Threshold conditions** for submitting requests (תנאי סף להגשת פניות)
- All available as PDFs from INGL coordination page

#### 3. Safety Buffer Zones
- High-pressure gas lines require **designated safety distances** from structures and facilities
- Exact distances depend on: pipe diameter, pressure (bar/psi), operating conditions, terrain
- Regulated by INGL safety appendices and international standards (ASME B31.8 — Gas Transmission and Distribution Piping Systems)
- Storage facilities likely require:
  - Larger buffer zones (due to risk profile)
  - Structural engineering assessment
  - Emergency response plan coordination

#### 4. Engineering Requirements for Adjacent Construction
- Any excavation, drilling, or ground disturbance within gas pipeline corridor:
  - Requires **engineering supervision** from INGL
  - **Structural reinforcement** may be required
  - **Monitoring** during and after construction
  - **Restoration** of pipeline corridor after work
- INGL maintains **24/7 emergency hotline**: 6778*

### INGL Infrastructure — Relevant Data

#### Pipeline System
- **Major segments**: Marine, Central, Southern, Northern, Eastern, Jerusalem line, duplication segments, export lines to Jordan (Sodom + North)
- **3 reception stations**: Ashdod, Ashkelon, Dor coast
- **Export lines**: Jordan (2 lines: Dead Sea + Beit She'an)
- Coverage: ~70% of Israel's electricity from gas via INGL

#### Key Customers
- **7 IEC power stations** + private producers (Dorad, Dalia, OPC, Adelteq, IPM, etc.)
- **Major industrial plants**: Bazan, Paz, Haifa Chemicals, Adama, Dead Sea Works
- **Export**: Egypt, Jordan

#### Tariff Structure (from INGL website — tabular data available)
- Capacity tariff (קיבולת)
- Flow tariff (הזרמה)
- Measurement discrepancy reports

### Additional Regulatory Context

#### 1. Ministry of Energy Regulation
- Fuel switching policy: coal → natural gas for power plants
- Rotenberg (2,200 MW) and Orot Rabin (1,100 MW) conversion underway
- Environmental regulations: methane emissions tracking (UNFCCC NIR reports)

#### 2. Environmental Requirements
- INGL operates under sustainability & ecological restoration commitments
- Coordination required with: KKL, Nature Reserves Authority, Stream Authorities
- Pre-construction: archaeological survey, ecological impact assessment

#### 3. Gas Market Reform
- Distribution licenses for local gas networks
- INGL is the monopoly transmission operator; distribution handled by separate licensees
- Storage: No large-scale gas storage facilities in Israel yet (different from electricity storage)

### Safety & Licensing Sources Available
1. **INGL coordination page** — PDFs for detailed planning guidelines
2. **INGL license appendices** — safety annexes downloadable from /holancha/
3. **State Comptroller report** on gas distribution network structure
4. **UNFCCC NIR report** — methane emissions, pipeline geometry data
5. **Ministry of Energy** — licensing database for gas facilities
6. **"Information for the Public" portal (CKAN/OData API)** — INGL workplace accidents database

### Practical Implications for a Storage Facility Near Gas Lines

To locate a storage facility near INGL infrastructure:
1. ✅ **Check proximity** — Use `grid_proximity(lon, lat)` to find nearest gas lines
2. ✅ **Submit coordination request** — Through national infrastructure coordination system
3. 📋 **Required documents**: Detailed engineering plan, safety assessment, buffer zone verification
4. ⚠️ **Risk factors**: Opposition keywords (קרינה, התנגד, מחאה) in protocol chunks within 15km
5. 📊 **Regulatory documents**: Search energy docs in the project for gas connection requirements

*Indexed: 2026-04-29 | Planning Intelligence Backend Audit*

### What is a Digital Engineering Design & Control Institute?

A **Digital Design & Control Institute** is an entity that provides automated, data-driven engineering design review and quality control services.

### Core Capabilities

#### 1. Automated Design Review
- **Clash detection** — Check if building systems conflict (pipes × beams × ducts × cables)
- **Code compliance checking** — Automatically verify designs against building codes, zoning, accessibility, fire safety
- **Standard compliance** — ISO 19650 (BIM), IFC (Industry Foundation Classes), national standards
- **Consistency checking** — Cross-disciplinary alignment (architecture × structure × MEP)

#### 2. Digital Twin Integration
- **Bidirectional data flow** between physical assets and virtual models
- **Real-time monitoring** from sensors, IoT, BIM
- **Predictive maintenance** — anticipate failures before they happen
- **Lifecycle management** — design → construction → operations → decommissioning

#### 3. Engineering Quality Control
- **Automated plan checking** — against statutory requirements
- **Cross-layer validation** — GIS (location) × BIM (design) × Regulations (code)
- **Issue tracking** — systematic error capture and resolution

### Key Software Tools (BIM & Design Review)
| Tool | Developer | Purpose |
|------|-----------|---------|
| Autodesk Revit | Autodesk | Core BIM modeling (buildings, infrastructure) |
| Autodesk Navisworks | Autodesk | Project review, clash detection, coordination |
| Autodesk BIM 360 | Autodesk | Cloud CDE — common data environment |
| Autodesk Tandem | Autodesk | Digital twin platform |
| Bentley iTwin | Bentley Systems | Infrastructure digital twin platform |
| Bentley MicroStation | Bentley | Infrastructure BIM modeling |
| Archicad | Graphisoft | Architectural BIM (macOS + Windows) |
| Allplan | Nemetschek | BIM for engineers and contractors |
| Tekla Structures | Trimble | Structural steel and concrete BIM |
| Solibri | Nemetschek | Model checking, code compliance, QA/QC |
| Revizto | Revizto SA | Issue tracking, coordination, VR review |
| Trimble Connect | Trimble | CDE and collaboration |
| FreeCAD / Bonsai | Open-source | Free BIM (Blender-based, IFC compatible) |
| Dynamo | Autodesk | Visual programming for Revit automation |

### International Standards
| Standard | What it governs |
|----------|-----------------|
| ISO 19650 | BIM information management — organization of design, construction, handover |
| ISO 16739 (IFC) | Data structure for BIM interoperability |
| ISO 12006 | Building construction — organization of building element classification |
| BS 1192 (UK) | BIM collaboration methodology (precursor to ISO 19650) |

### Israel-Specific Context
- **Israeli BIM Standard** — based on ISO 19650, adapted by Israeli Standards Institute
- **Admin for Planning** — moving toward digital submission (XPLAN, iplan.gov.il)
- **Government BIM mandate** — gradually requiring BIM for public projects
- **iplan.gov.il** — national planning portal

### How This Connects to Your Planning Intelligence Project
Your project already has **GIS + Data Intelligence layer**. A Digital Design & Control Institute would add:
1. **Design review** — Validate plans against zoning, land use, infrastructure constraints
2. **Code compliance** — Automated checking against building codes and statutory requirements
3. **Cross-layer validation** — Do GIS data (location, infrastructure) match BIM data (design)?
4. **Integration** — Connect site scores, proximity, opposition risk with BIM models
5. **Control tower** — Centralized oversight: design → permit → construction → operation


### Gap Analysis
| Have | Missing |
|------|---------|
| GIS data (35K plans, grid, gas, land) | No BIM integration (IFC import) |
| Site intelligence (synergy score, opposition) | No design review engine |
| Regulatory knowledge (42 energy docs) | No automated code compliance rules |
| Committee KPIs (136) | No cross-committee workflow tracking |
| Protocol data (19K chunks) | No structured requirements database |

---

## Project Requirements — Core Checklist for Planning Permits (Lior's Training Data)

### 1. Local Planning & Building Committee (ועדה מקומית) — Core Requirements
- Complete architectural plans: approved state, proposed state, sections, elevations
- Updated survey signed by licensed surveyor
- Parking & internal traffic appendix
- Floor area table (main / service / employment)
- Zoning compliance check + declaration of adherence
- Drainage & runoff management appendix
- Applicant's letter of commitment
- Land ownership / rights documents

> Almost always: requests for clarifications, graphical corrections, or supplements.

### 2. Municipal Engineer (מהנדס הוועדה / רשות מקומית)
- Parking and usage calculations
- Compatibility with existing road network
- Entrance/exit points to lot
- Signature on compatibility with public infrastructure
- Sometimes: commitment to perform development works

### 3. Traffic/Transportation Consultant
- Traffic & parking appendix
- Traffic counts (if required)
- Traffic load approval and operational solutions
- Connection to existing road system

### 4. Fire & Rescue Services (כבאות והצלה)
Almost always required:
- Fire safety appendix
- Fire truck access routes plan
- Safety distances between buildings
- Water supply positions / fire hydrants
- Fire safety consultant declaration
- Sometimes: conditions for permit subject to execution

### 5. Ministry of Environmental Protection / City Union
Common requirements:
- Environmental appendix
- Acoustic assessment (noise)
- Odors / dust / emissions document
- Work process description and environmental impact
- Determination: business license / emission permit / restrictive conditions

### 6. Water & Sewage Corporation
- Water & sewage appendix
- Flow calculations
- Pre-treatment solutions (grease / leachate)
- Connection point approval
- Sometimes: letter of commitment or fees

### 7. Electricity / Energy
- IEC expert opinion
- Transformer requirement / transformer room
- Existing infrastructure locations
- Route coordination

### 8. Accessibility (mandatory in almost every project)
- Accessibility appendix (TAMAS)
- Accessibility consultant declaration
- Parking, pathways, restroom solutions
- Dedicated signature as part of the permit

### 9. Business Licensing / Health (sometimes parallel to permit)
- Purpose and usage description
- Health department opinion
- Waste management arrangements
- Health bureau approval (for waste/industry)

### 10. Supplementary Bodies (by location)
- Israel Antiquities Authority — survey / commitment
- Nature & Parks Authority / KKL — if near open space
- Netivei Israel / NTA — proximity to national road
- Israel Land Authority (RMI) — authorization documents

### Key Insight
> 80% of delays stem from environmental, fire, and traffic requirements — not from architecture.

---

*Indexed: 2026-04-29 | Planning Intelligence Backend Audit*

