# 🚖 Uber Trip Analysis — Power BI Project

Interactive analysis of **Uber trip data** built in **Power BI** using **Excel** as the data source. The report contains data modeling, DAX measures, calculated columns, and three focused report pages: **Overview Analysis**, **Time Analysis**, and **Details**. Screenshots provided in the `assets/` folder illustrate the report pages and the model's Measures/Tables list as seen in Power BI Desktop.

---

## ⭐ Highlights (What this report delivers)

- Executive KPIs (Total bookings, booking value, trip distance, averages).  
- Payment type and trip-type breakdowns (donut charts).  
- Day vs Night comparison and counts.  
- Location analysis: most frequent pickup/dropoff points, top-5 locations, and the farthest trip (text label).  
- Vehicle-level summary table (Total Booking, Total Booking Amount, Avg Booking Value, Total Trip Distance).  
- Time patterns: hourly pickup value trend, heatmap (hour × weekday), and dynamic metric toggles.  
- AI visuals: Decomposition Tree and Key Influencers (to detect drivers of booking value).

---

## 🗂 Project structure (suggested)

```
Uber-Trip-Analysis/
├─ data/
│  └─ uber_trips.xlsx                    # Excel data source used in Power BI
├─ pbix/
│  └─ Uber Trip Analysis.pbix            # Power BI report file
├─ assets/
│  ├─ overview-analysis.png
│  ├─ time-analysis.png
│  └─ details.png
└─ README_EN.md
```

Place the screenshots (from the images you've shared) inside `assets/` with the names above to embed them in the README if needed.

---

## 📁 Data source & tables (as in the report)

**Source:** Excel file (loaded through Power Query in Power BI).

### Main tables
1. **Trip Details** (main fact table) — important fields visible in your screenshots:
   - `Trip ID`, `Pickup Date`, `Pickup Time`, `Pickup Hour`, `Pickup Hour (bins)`
   - `Drop Off Time`, `Trip Duration (Minutes)`
   - `trip_distance`, `fare_amount`, `Surge Fee`, `fare_total` (or combined)
   - `Payment_type`, `Vehicle`, `Trip Type`, `Trip Category`
   - `passenger_count`, `Passenger Group`
   - `PULocationID`, `DOLocationID`, `PULocation`, `DOLocation` (if available)
2. **Location Table**:
   - `LocationID`, `Location`, `City` (used for pickup and dropoff names)
3. **Calendar Table**:
   - `Date`, `Day Name`, `Year`, `Month`, `Weekday`, etc.
4. **Dynamic Measures** (disconnected helper tables):
   - `Dynamic Measures`, `Dynamic Measures Fields`, `Dynamic Measures Order`, `Dynamic Title` (tables to support metric switching and dynamic text).

> The right-side panes in your screenshots show many of these table names and measures; include exact names from your model when finalizing the pbix to keep the README 1:1 with your file.

---

## 🔗 Data model design & relationships

- `Trip Details[Pickup Date]` **→** `Calendar[Date]` (active relationship) for time slicing.  
- `Trip Details[PULocationID]` **→** `Location[LocationID]` (active) for pickup analysis.  
- `Trip Details[DOLocationID]` **→** `Location[LocationID]` (inactive) — used with `USERELATIONSHIP()` inside measures when analyzing dropoff.  
- Disconnected `Dynamic Measures` tables used for metric selection in visuals (a classic pattern for toggles).

This design allows one `Location` table to serve both pickup and dropoff contexts and uses USERELATIONSHIP to switch contexts inside measures without duplicating location tables.

---

## 🧮 Key DAX Measures (full examples)

> Use the exact field names in your model if they differ — adjust names accordingly.

### Basic totals & aggregates
```DAX
Total Booking = 
COUNTROWS('Trip Details')

Total Booking Amount = 
VAR Fare  = SUM('Trip Details'[fare_amount])
VAR Surge = COALESCE(SUM('Trip Details'[Surge Fee]), 0)
RETURN Fare + Surge

Total Trip Distance = 
SUM('Trip Details'[trip_distance])
```

### Averages & typical KPIs
```DAX
Avg Booking Value = 
DIVIDE([Total Booking Amount], [Total Booking])

Avg Trip Distance = 
DIVIDE([Total Trip Distance], [Total Booking])

Avg Trip Time = 
AVERAGE('Trip Details'[Trip Duration (Minutes)])
```

### Day vs Night split
```DAX
Day Trips = 
CALCULATE([Total Booking], 'Trip Details'[Trip Type] = "Day Trip")

Night Trips = 
CALCULATE([Total Booking], 'Trip Details'[Trip Type] = "Night Trip")
```

### Min / Max trip duration
```DAX
Max Trip Duration = MAX('Trip Details'[Trip Duration (Minutes)])
Min Trip Duration = MIN('Trip Details'[Trip Duration (Minutes)])
```

### Pickup vs Dropoff totals (activate inactive relationship when needed)
```DAX
-- Uses the active relationship (Pickup)
Total Booking (Pickup) = [Total Booking]

-- Uses the inactive relationship (Dropoff)
Total Booking (Dropoff) =
CALCULATE(
    [Total Booking],
    USERELATIONSHIP('Trip Details'[DOLocationID], 'Location'[LocationID])
)
```

### Most frequent pickup & dropoff points (text labels)
```DAX
Most Frequent Pickup Point =
VAR T = ADDCOLUMNS(VALUES('Location'[Location]), "Bookings", [Total Booking (Pickup)])
VAR Top1 = TOPN(1, T, [Bookings], DESC)
RETURN CONCATENATEX(Top1, 'Location'[Location], ", ")

Most Frequent Dropoff Point =
VAR T = ADDCOLUMNS(
    CALCULATETABLE(VALUES('Location'[Location]), USERELATIONSHIP('Trip Details'[DOLocationID], 'Location'[LocationID])),
    "Bookings", [Total Booking (Dropoff)]
)
VAR Top1 = TOPN(1, T, [Bookings], DESC)
RETURN CONCATENATEX(Top1, 'Location'[Location], ", ")
```

### Farthest trip (text label showing pickup → dropoff and distance)
```DAX
Farthest Trip =
VAR MaxRow = TOPN(1, ALL('Trip Details'), 'Trip Details'[trip_distance], DESC)
VAR PU = MAXX(MaxRow, RELATED('Location'[Location]))  -- via Pickup relationship
VAR DO = MAXX(MaxRow, CALCULATE(RELATED('Location'[Location]), USERELATIONSHIP('Trip Details'[DOLocationID],'Location'[LocationID])))
VAR Dist = MAXX(MaxRow, 'Trip Details'[trip_distance])
RETURN "Pickup: " & PU & " → Drop-off: " & DO & " (" & FORMAT(Dist, "#,0.0") & " miles)"
```

### Dynamic metric selector (switch between Booking Amount / Bookings / Trip Distance)
```DAX
Selected Metric =
VAR Metric = SELECTEDVALUE('Dynamic Measures Fields'[Metric])
RETURN
SWITCH(TRUE(),
    Metric = "Total Booking Value", [Total Booking Amount],
    Metric = "Total Bookings",      [Total Booking],
    Metric = "Total Trip Distance", [Total Trip Distance],
    BLANK()
)
```

### Dynamic report title (example)
```DAX
Dynamic Title =
VAR CityName = SELECTEDVALUE('Location'[City], "All Cities")
VAR DateTxt = FORMAT(MIN('Calendar'[Date]), "d/M/yyyy") & " → " & FORMAT(MAX('Calendar'[Date]), "d/M/yyyy")
RETURN "Uber Trip Analysis — " & CityName & " | " & DateTxt
```

---

## 🧱 Calculated columns (examples)

Use calculated columns when you need row-level classification (but favor measures for aggregated logic when possible).

```DAX
-- Distance category
Distance Category =
VAR d = 'Trip Details'[trip_distance]
RETURN SWITCH(TRUE(),
    d < 3, "Short Trip",
    d < 8, "Medium Trip",
           "Long Trip"
)

-- Trip Type by hour (Day / Night)
Trip Type =
VAR h = 'Trip Details'[Pickup Hour]
RETURN IF(h >= 6 && h < 18, "Day Trip", "Night Trip")

-- Pickup Hour bins (categorical buckets for the heatmap)
Pickup Hour (bins) =
VAR h = 'Trip Details'[Pickup Hour]
RETURN SWITCH(TRUE(),
    h < 3,  "12:00 AM – 3:00 AM",
    h < 6,  "3:00 AM – 6:00 AM",
    h < 9,  "6:00 AM – 9:00 AM",
    h < 12, "9:00 AM – 12:00 PM",
    h < 15, "12:00 PM – 3:00 PM",
    h < 18, "3:00 PM – 6:00 PM",
    h < 21, "6:00 PM – 9:00 PM",
           "9:00 PM – 12:00 AM"
)
```

### Calendar table (example)
```DAX
Calendar =
ADDCOLUMNS(
    CALENDAR(DATE(2024,6,1), DATE(2024,6,30)),
    "Day Name", FORMAT([Date], "ddd"),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMM"),
    "WeekdayNumber", WEEKDAY([Date],2)
)
```

---

## 📊 Report pages & visuals (detailed)
The three main report pages replicate what appears in your screenshots and include the following visuals and layout patterns.

### 1) Overview Analysis (page)
- **KPIs / Left column cards:** Total Booking, Total Booking Amount, Total Trip Distance, Avg Trip Distance, Avg Booking Value, Avg Trip Time, Day Trips, Night Trips.  
- **Donut visuals:** Total Booking by `Payment_type`, Total Booking by `Trip Type`.  
- **Line chart / Area chart:** Total Booking by Day (monthly days).  
- **Vehicle Type Analysis table:** Row-level aggregation by `Vehicle` showing Total Booking, Total Booking Amount, Avg Booking Value, Total Trip Distance.  
- **Location Analysis card:** Most Frequent Pickup Point, Most Frequent Dropoff Point, Farthest Trip (text).  
- **Top 5 Location bar chart** and **Most Preferred Vehicle for Location Pickup** (bar chart).  
- **AI visuals:** Decomposition Tree + Key Influencers to explore drivers of booking value.  

### 2) Time Analysis (page)
- **Dynamic metric selector** (buttons or disconnected table slicer): toggles between `Total Booking Value`, `Total Bookings`, `Total Trip Distance`.  
- **Bar chart:** Total Booking Value by Vehicle (or by time bucket).  
- **Donut charts:** Booking Value by Distance Category, Booking Value by Passenger Group.  
- **Area/Line chart:** Total Booking Value by Pickup Time (detailed hourly series).  
- **Heatmap:** `Total Booking Value by Day Name` × `Hour` (the heatmap uses `Selected Metric` to change metric on the fly).  
- **Small multiples / side histograms** for dynamic metric.  

### 3) Details (page)
- **Table visual** containing Trip ID, Pickup Date, Vehicle, Payment_type, NO. OF. Passenger, Total Trip Distance, Booking Value, Pickup Location, Pickup Hour, Trip Type, and other fields; with a totals row at the bottom. This is used for data validation and drill-through from other pages.

---

## 🎛 Filters & Slicers used
- **Date Range** slicer (start → end), used across pages.  
- **City** slicer.  
- **Dynamic Metric** slicer (from `Dynamic Measures Fields` table).  
- Keep all filters option used (UI setting) to make visuals respect slicers consistently.  
- Cross-report drill-through is shown as Off in screenshots; drill-through to Details page is available for inspected rows.

---

## 🧰 Performance & best practices implemented / recommended
- Use Measures rather than heavy calculated columns for aggregations where possible to reduce model size.  
- Use `VAR` inside DAX to avoid recalculating expressions.  
- Disable Auto Date/Time in Power BI and prefer a custom Calendar table.  
- Use `USERELATIONSHIP()` only when necessary to avoid unnecessary context switching.  
- Use **Performance Analyzer** to capture slow visuals, and optimize visuals (avoid too many complex visuals on a single page).  
- Ensure numeric formatting is consistent (currency, distances, minutes).  
- Consider aggregations or composite models if the dataset grows beyond Power BI desktop comfortable limits.

---

## ✅ Business questions answered by this report
- What is the **total booking volume** and **total booking value** for the selected period?  
- What is the **average booking value**, **avg trip distance**, and **avg trip duration**?  
- When do bookings peak during the day and week? (hourly & heatmap insights)  
- Which **pickup/dropoff locations** are most frequent and which trip is the farthest?  
- Which **vehicle types** generate the most bookings and booking value?  
- How do **day vs night** trips differ in volume and value?  
- What payment methods are most used?

---

## 🚀 How to run the report (step-by-step)
1. Open `pbix/Uber Trip Analysis.pbix` in **Power BI Desktop**.  
2. In **Transform Data → Data Source Settings**, update the path to `data/uber_trips.xlsx` if needed.  
3. Click **Refresh** to load data.  
4. Use slicers (Date, City, Dynamic Metric) to explore.  
5. Open **Performance Analyzer** (View → Performance Analyzer) to capture runtime if you need optimization traces.

> Recommended: Power BI Desktop (June 2024 or later) for full feature parity with the visuals used (AI visuals + decomposition tree).

---

## 📌 Licensing & notes
- Data used for demonstration/educational purposes.  
- This README and sample DAX snippets licensed under **MIT** unless otherwise requested.  

---

## 👤 Author & contact
**Mahmoud Elzeiat** — Data Analyst  
- Email: mahmoudelzeiat7@gmail.com  
- Phone: +20 01044293980  
- LinkedIn: https://www.linkedin.com/in/mahmoud-elzayat-data-analysis
