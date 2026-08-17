# Crane Resort — Occupancy & Revenue Dashboard

Power BI dashboard analyzing synthetic booking data shaped like The Crane Resort's
actual operations — room types, seasonality (Barbados high/low season), and
booking channel mix.

**Status:** Data ready → build in Power BI Desktop → export screenshots

---

## Data Model (Star Schema)

```
dim_room_type.csv ──┐
                     ├──> fact_bookings.csv <──── dim_date.csv
                     │    (booking_id, room_type,
                     │     check_in, check_out, nights,
                     │     nightly_rate, total_revenue,
                     │     status, booking_channel)
```

- `dim_date.csv` — 365 days (2026), includes `Season` (High: Dec-Apr, Low: May-Nov) — Barbados' actual tourism seasonality
- `dim_room_type.csv` — 4 room types with real inventory counts (needed for Occupancy Rate — can't calculate it from revenue alone)
- `fact_bookings.csv` — 1,732 bookings, `status` includes Cancelled (don't count these in revenue), `booking_channel` for direct-vs-OTA analysis

---

## Build Steps (Power BI Desktop)

### 1. Import data
- Get Data → Text/CSV → import all three files
- Model view → drag relationships:
  - `fact_bookings[room_type]` → `dim_room_type[room_type]`
  - `fact_bookings[check_in]` → `dim_date[Date]`
- Right-click `dim_date` → **Mark as date table** (required for time intelligence functions)

### 2. Core DAX measures
Create these in a new table (Modeling → New Table → name it `_Measures`, keeps them out of the data tables):

```dax
Total Revenue =
CALCULATE(
    SUM(fact_bookings[total_revenue]),
    fact_bookings[status] = "Completed"
)

Room Nights Sold =
CALCULATE(
    SUM(fact_bookings[nights]),
    fact_bookings[status] IN {"Completed", "Confirmed"}
)

Available Room Nights =
SUMX(
    dim_room_type,
    dim_room_type[total_rooms] * DATEDIFF(MIN(dim_date[Date]), MAX(dim_date[Date]), DAY)
)

Occupancy Rate =
DIVIDE([Room Nights Sold], [Available Room Nights])

ADR (Average Daily Rate) =
DIVIDE([Total Revenue], [Room Nights Sold])

RevPAR =
DIVIDE([Total Revenue], [Available Room Nights])

Direct Booking % =
DIVIDE(
    CALCULATE([Total Revenue], fact_bookings[booking_channel] = "Direct Website"),
    [Total Revenue]
)

Cancellation Rate =
DIVIDE(
    CALCULATE(COUNTROWS(fact_bookings), fact_bookings[status] = "Cancelled"),
    COUNTROWS(fact_bookings)
)
```

**Why these specific metrics:** ADR and RevPAR aren't arbitrary — they're the two
numbers every hotel revenue manager actually tracks. ADR tells you pricing power;
RevPAR tells you pricing power *combined with* how full you are. A high ADR with
low occupancy is a worse business than a lower ADR that's fully booked.

### 3. Visuals to build
- **Top row — KPI cards:** Total Revenue, Occupancy Rate, ADR, RevPAR (use Card visual)
- **Line chart:** Total Revenue by Month (`dim_date[MonthName]` on axis) — shows the seasonal swing
- **Clustered bar:** Total Revenue by Room Type
- **Donut chart:** Revenue by Booking Channel — visualizes direct-vs-OTA mix
- **Matrix/heatmap:** Occupancy Rate by Room Type (rows) × Month (columns)
- **Slicers:** Season (High/Low), Room Type, Booking Channel

### 4. Save
- File → Save As → `crane-occupancy-revenue.pbix`
- File → Export → PDF or use Snipping Tool for a PNG screenshot of the finished dashboard (for the GitHub README — GitHub can't render `.pbix` directly)

---

## What this demonstrates (for your portfolio / internal transfer case)
- Star-schema data modeling, not just a flat spreadsheet
- Real hospitality KPIs (Occupancy, ADR, RevPAR) — not generic "sales dashboard" filler
- Seasonality-aware analysis specific to Barbados tourism patterns
- Direct booking share — a metric that maps straight to a real cost-saving conversation (OTAs take commission, direct bookings don't)
