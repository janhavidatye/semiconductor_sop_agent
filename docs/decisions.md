# Decision log


Record of every locked decision and the reason behind it. Newest decisions are added at the bottom.

---

## 1. Project direction (Nov 2026)

| Decision | Reason |
|---|---|
| Build an S&OP **supply review** tool, not the full S&OP process | Keeps scope realistic; demand review, pre-S&OP and executive meeting are future work |
| Inspired by Ryan Wahl's `sop-cycle-agent`, but all code written from scratch | His repo has no license, so the code cannot be reused; ideas are credited in the README |
| Design rule: **code decides, AI explains and judges, code checks the AI** | AI makes arithmetic mistakes, gives different answers on reruns and cannot be audited; code is exact, repeatable and testable |
| 3 AI agents: Analyst, Exception, Reviewer | Each does work code cannot (writing, judging trade-offs, checking meaning); more agents add cost and failure points |
| Add 12-month simulation and a look-back on past recommendations | Fixes the main gaps in the original project: one cycle only, self-graded answer key, no feedback loop |
| Dashboards and a user interface are in scope | Makes results visible to recruiters and planners; tool choice to be decided later |
| MIT license | Lets others reuse the code legally |

## 2. Product type (Nov 2026)

| Decision | Reason |
|---|---|
| **Semiconductor fab consumables** | Strong local employers (TI, Samsung, GlobalWafers, KLA); real supplier risk; a product I have not worked on before |
| Material groups: **Wafers, Lithography, Deposition, CMP** | Together they cover every material type the safety stock rules need |
| Gases and wet chemicals left out | Gases overlap with past work at Air Products; four groups are enough |
| Add a **shelf-life rule**: safety stock for expiring items cannot exceed what can be used before expiry | Photoresist and slurries expire; the original project ignored this |

## 3. Locked materials (16)

Lead time: Short = under 4 weeks, Med = 4-8 weeks, Long = over 8 weeks. Labels are rough and must be researched before results are reported.

| # | Material | Group | Type | Supplier country | Lead time | Cost | Critical | Expires |
|---|---|---|---|---|---|---|---|---|
| 1 | 300mm prime wafer | Wafers | Steady | Japan | Long | Med | Yes | No |
| 2 | 300mm epitaxial wafer | Wafers | Steady | Taiwan | Long | High | Yes | No |
| 3 | Test / monitor wafer | Wafers | Steady | Korea | Med | Low | No | No |
| 4 | SOI wafer (specialty) | Wafers | Custom | France | Long | High | No | No |
| 5 | ArF photoresist | Lithography | Steady | Japan | Long | High | Yes | Yes |
| 6 | KrF photoresist | Lithography | Steady | Japan | Med | Med | Yes | Yes |
| 7 | Developer (TMAH) | Lithography | Steady | USA | Short | Low | No | No |
| 8 | Photomask | Lithography | Custom | Taiwan | Med | High | Yes | No |
| 9 | Copper sputtering target | Deposition | Wear | USA | Med | Med | Yes | No |
| 10 | Tantalum sputtering target | Deposition | Wear | Japan | Long | High | Yes | No |
| 11 | Titanium sputtering target | Deposition | Wear | USA | Med | Med | No | No |
| 12 | ALD precursor (hafnium) | Deposition | Steady | Germany | Long | High | Yes | Yes |
| 13 | Copper CMP slurry | CMP | Steady | USA | Med | Med | Yes | Yes |
| 14 | Oxide (ceria) CMP slurry | CMP | Steady | Japan | Med | Med | No | Yes |
| 15 | CMP polishing pad | CMP | Wear | USA | Med | Med | Yes | No |
| 16 | Pad conditioner disc | CMP | Wear | Korea | Short | Low | No | No |

## 4. Data design (Nov 2026)

Three design ideas:

1. **Demand comes from wafer starts.** Demand = actual wafer starts x usage per wafer + random noise + fab events. This is how real fabs plan consumables.
2. **Spikes come from fab events**, not customer orders: production ramps, new product launches and tool maintenance.
3. **Inventory is tracked by lot with expiry dates**, so the shelf-life rule can work. Lots are used oldest-expiry first (FEFO).

Ten tables, all stored in SQLite:

| # | Table | Columns |
|---|---|---|
| 1 | Materials | material_id (MATNR), description (MAKTX), material_group (MATKL), demand_type, unit_of_measure (MEINS), supplier_id (LIFNR), lead_time_days (PLIFZ), unit_cost_usd (STPRS), lot_size (BSTRF), shelf_life_days (MHDHB), critical, usage_per_wafer, current_safety_stock (EISBE), qc_hold_days |
| 2 | Suppliers | supplier_id (LIFNR), supplier_name (NAME1), country (LAND1), risk_tier, risk_reason, on_time_delivery_pct |
| 3 | Supplier tier history | change_id, supplier_id, change_date, old_tier, new_tier, reason |
| 4 | Wafer starts | week_start_date, planned_wafer_starts, actual_wafer_starts |
| 5 | Demand history | week_start_date, material_id, qty_used |
| 6 | Demand forecast | forecast_date, week_start_date, material_id, stat_forecast, consensus_forecast, confidence_pct |
| 7 | Fab events | event_id, event_type, material_id, start_week, end_week, extra_qty, known_in_advance, announced_date |
| 8 | Inventory lots | lot_id (CHARG), material_id, qty_on_hand, received_date, expiry_date, status |
| 9 | Purchase orders | po_id (EBELN), material_id, supplier_id (LIFNR), order_qty, order_date (BEDAT), due_date (EINDT), received_date, po_type, status |
| 10 | Recommendations log | rec_id, cycle_date, material_id, rec_type, rec_qty, rule_applied, problem_targeted, outcome, outcome_notes |

Key rules agreed with the tables:

| Decision | Reason |
|---|---|
| SAP MM field names used where they exist | Makes the data look like a real ERP export; sets up the messy-data step |
| Wear items use usage per wafer (e.g. a pad lasting 2,000 wafers = 0.0005 per wafer) | No extra column needed |
| Custom items leave usage per wafer blank; demand comes only from fab events | Photomasks and SOI wafers are not used per wafer |
| All supplier names are fictional | Avoids making claims about real companies |
| 6-10 supplier tier changes over the simulated year | Enough to test the 30-day rule |
| One wafer starts total for the whole fab | Simpler; a split by product can be added later |
| 78 weeks of history, 52 weeks ahead | Enough to measure variability and run 12 months |
| Expired material is scrap, not demand | Counting it as demand would inflate forecasts for items that expire |
| Every forecast is saved with its date | Needed to measure forecast accuracy later |
| confidence_pct for custom items only (buy at 70% or higher) | Supports the custom-item buying rule |
| Ramp events do not add extra_qty | Ramps already raise wafer starts; avoids counting demand twice |
| Keep ON_HOLD status with fixed QC hold days per material | Separates stock on the shelf from stock that can be used |
| No backlog netting | A fab has no customer backlog for consumables |
| Supplier stored on each PO | Allows alternate-supplier orders |
| Look-back outcomes decided by code: WORKED, FAILED, NOT_NEEDED, PENDING | Same principle as the rest of the project; NOT_NEEDED is an estimate, stated honestly |
| All recommendations assumed accepted in the simulation | Tracking planner rejections adds complexity with little benefit |

## 5. Build order (Nov 2026)

| Decision | Reason |
|---|---|
| Build with clearly labelled placeholder numbers first, research real numbers later | Almost all researched numbers live in the Materials table, so they can be swapped in one place |
| **No results in the README, resume or interviews until real numbers are researched** | Prevents presenting made-up numbers as findings |
| Repo: `semiconductor_sop_agent` | Set up Nov 9, 2026 with README, MIT license, Python .gitignore and a virtual environment |

## Open items

- Research real lead times, costs, shelf lives, usage per wafer and QC hold days (parked)
- Choose dashboard and user interface tools
- Decide which fields to make messy on purpose
