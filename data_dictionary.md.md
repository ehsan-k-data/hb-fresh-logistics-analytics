# Data Dictionary: HB Fresh Logistics

All data is synthetic and covers **1 Nov 2025 – 31 Oct 2026**. Currency is AUD. Distances are in kilometres.

---

## Fact tables

### fact_manifest: one row per trip (5,000 rows)

| Column | Type | Description |
|---|---|---|
| Manifest | Integer | Unique trip ID (primary key) |
| Vehicle | Text | Vehicle registration. Joins to `dim_vehicle[Registration]`. "SUBCON" for subcontracted trips |
| Fleet Code | Text | Internal fleet code (V101, RENTAL, SUBCON…) |
| ETA | Date | Arrival date (inactive relationship to `dim_date`) |
| ETD | Date | Departure date (**active** relationship to `dim_date`) |
| Route | Text | Origin-destination code, e.g. `HFM-HFA`. Joins to `dim_route` |
| Route Band | Text | Linehaul, Long Haul, Regional or Metro |
| Klm | Integer | Trip distance (km) |
| Driver | Text | Driver code. Blank for subcontracted trips. Joins to `dim_driver` |
| Subby | Text | Subcontractor code. Blank for own-fleet trips. Joins to `dim_subcontractor` |
| Origin / Destination | Text | Full location names |
| Items | Integer | Number of consignment items |
| Weight | Decimal | Total weight (kg) |
| Volume | Decimal | Total volume (m³) |
| Amount | Currency | Trip revenue |
| Driver Cost | Currency | Driver wages for the trip (0 for subcontracted trips) |
| Subby Cost | Currency | Amount paid to the subcontractor (0 for own-fleet trips) |
| Truck Cost | Currency | Vehicle running cost: fuel, maintenance, lease or rental (0 for subcontracted trips) |
| **Fleet Source** *(calculated)* | Text | "Subcontractor" if Subby Cost > 0, otherwise "Own Fleet" |

### fact_leg: one row per delivery or drop (11,479 rows)

| Column | Type | Description |
|---|---|---|
| Leg No | Integer | Unique leg ID (primary key) |
| Manifest | Integer | Parent trip. Each trip has one or more legs |
| Date | Date | Leg date (active relationship to `dim_date`) |
| Docket | Integer | Consignment docket number |
| Customer | Text | Customer code. Joins to `dim_customer` |
| Fleet Code | Text | Fleet code of the vehicle |
| RouteFrom / RouteTo | Text | Pickup and delivery location codes (split from the source Route). RouteFrom is active to `dim_location`, RouteTo is inactive |
| Route *(rebuilt)* | Text | `RouteFrom-RouteTo`. Joins to `dim_route` |
| Sender / Receiver | Text | Full location names |
| Trailer | Text | Trailer ID (mostly blank) |
| Driver | Text | Driver code (blank for subcontracted legs) |
| Subby | Text | Subcontractor code (blank for own-fleet legs) |
| Vehicle Revenue | Currency | Revenue for this leg. Legs add up exactly to trip revenue ($5,364,407.77) |
| Klm | Integer | Distance attributed to the leg |
| Pallets | Decimal | Pallets delivered |

---

## Dimension tables

### dim_date (calculated)
| Column | Description |
|---|---|
| Date | Calendar date (key) |
| Year, Month No, Month | Calendar parts |
| YearMonth | Sort key, e.g. 202511 |
| Month Year | Label, e.g. "Nov 2025" (sorted by YearMonth) |
| Weekday, Weekday No | Day name and number, Monday = 1 (Weekday sorted by Weekday No) |
| Week Start | Monday of the week |

### dim_route (20 rows)
| Column | Description |
|---|---|
| Route | Route code (key), e.g. `HFS-VGS` |
| Route Band | Linehaul, Long Haul, Regional or Metro |

### dim_vehicle (34 rows)
| Column | Description |
|---|---|
| Registration | Plate number (key) |
| Fleet Code | V1xx = prime movers, V2xx = rigids, V3xx = refrigerated vans, RENTAL, SUBCON |
| Vehicle Type | Prime Mover, Rigid (12 pallet), Refrigerated Van or Subcontractor |
| Ownership | Owned, Rental or Subcontractor |
| Primary Use | Linehaul, Metro/Regional, Metro or Any |
| **Vehicle Label** *(calculated)* | `Fleet Code · Registration`, to tell the rental trucks apart |

### dim_driver (45 rows)
| Column | Description |
|---|---|
| Driver Code | Key |
| Driver Name | Fictional name |
| Base Depot | Melbourne, Adelaide or Sydney |
| Driver Role | Linehaul or Metro/Regional |

### dim_customer (40 rows)
| Column | Description |
|---|---|
| Customer Code | Key |
| Customer Name | Fictional name |
| Segment | Produce, Dairy, Meat, Ready Meals, Dry Goods, Seafood, Bakery, Beverages, Frozen or Floral |

### dim_location (19 rows)
| Column | Description |
|---|---|
| Location Code | Key, e.g. HFM |
| Location Name | Full name |
| Location Type | Depot, Retail DC, Cold Store or Grower |
| Suburb, State | Location |

### dim_subcontractor (6 rows)
| Column | Description |
|---|---|
| Subcontractor Code | Key |
| Subcontractor Name | Fictional carrier name |

---

## Key measures

| Measure | Definition |
|---|---|
| Revenue | `SUM(fact_manifest[Amount])` (trip-level) |
| Leg Revenue | `SUM(fact_leg[Vehicle Revenue])`. Use with customer or segment |
| Total Cost | Driver Cost + Subby Cost + Truck Cost |
| Profit | Revenue − Total Cost. **Excludes overheads** |
| Margin % | Profit ÷ Revenue |
| Total Manifests / Total Legs | Row counts of each fact table |
| Total Km | `SUM(fact_manifest[Klm])` |
| Revenue / Cost / Profit per Km | Measure ÷ Total Km |
| Revenue per Pallet | Leg Revenue ÷ Total Pallets |
| Subby Revenue / Own Fleet Revenue | Revenue filtered by Fleet Source |
| Subby Share % | Subby Revenue ÷ Revenue |
| Own Fleet Margin % / Subby Margin % | Margin % filtered by Fleet Source |
| Revenue MoM % | Month-on-month change using `DATEADD` |
| Active Routes / Customers / Drivers, Fleet Size | Distinct counts |
| Driver Days Worked | Distinct driver-and-date combinations |
| Driver Cost % of Revenue | Driver Cost ÷ Revenue |
| Legs by Destination | Leg count through the inactive RouteTo relationship (`USERELATIONSHIP`) |
| Avg Monthly Revenue, Peak / Lowest Month, Busiest Day | Seasonality cards (`AVERAGEX`, `TOPN`) |

## Notes and limitations
- Costs are recorded per trip, not per customer, so customer analysis uses revenue and revenue per pallet, not profit.
- Profit is operating contribution. Depot, admin and other overheads are not in the data.
- Subcontracted trips have no driver, so filtering by driver or depot removes them.
