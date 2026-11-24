# 02. Source inventory and mapping


## 1. Source inventory

| Source | Purpose | Origin | Rows | Candidate key | Format | Refresh assumption |
|---|---|---|---|---|---|---|
| `ica.csv` | Insured loss and claims per catastrophe event (main facts) | Insurance Council of Australia | 747 events, 1967–2026 | None reliable. `CAT Name` is shared by 497 "Undeclared" rows, so the key is the source row number | CSV, 3 blank rows above the header, 23 columns | Updated after each declared event |
| `disaster_events.csv` | Deaths, injuries, evacuations and homes destroyed per event | data.gov.au | 673 events, 1829–2014 | `id` | CSV, header on row 1, 43 columns | Frozen since 2014 |
| `abs_population.csv` | Estimated resident population by state and year | ABS Data by region | 26,188 region-year rows | (`Code`, `Year`) | CSV, 6 title rows (two are `#REF!`) | Annual |
| `abs_income.csv` | Median income by state and year | ABS Data by region | 32,006 region-year rows | (`Code`, `Year`) | CSV, 6 title rows, repeated income-band headers | Annual |

**Note on ICA:** ICA is industry data pooled by the Insurance Council, not Kestrel's own claims. We treat it as the client's industry claims feed. This is a framing choice and is stated under limitations in the report.

## 2. Mapping table

| Source field | Staging column | Warehouse column | Rule |
|---|---|---|---|
| ICA `CAT Name` | `cat_name` | `dim_event.ica_cat_code` | Uppercase and trim. Not used as a key (DQ4) |
| ICA row number | `_row_num` | `dim_event.source_row_num` | Event key comes from the source row |
| ICA `Event Name` | `event_name` | `dim_event.event_name` | Trim |
| ICA `Event Start` / `Event Finish` | `event_start` / `event_finish` | `fact_event_loss.start_date_key` / `end_date_key` | Parse `d-MMM-yy`. If the year is after the current year, subtract 100 years (DQ1). Look up `dim_date`; unmatched goes to key -1 |
| ICA `State` | `state` | `dim_location.location_key` and `dim_event.state_original` | Normalise `and` and `&` to commas, take the first state, set `multi_state_flag`. Map through `xw_state` (FNQ, SEQ, SWQ → QLD). If blank, infer from event name and town via `xw_state_keyword` and set `state_source = 'inferred'`. Otherwise Unknown (-1) (DQ2) |
| ICA `Type` | `type` | `dim_hazard.hazard_group` and `hazard_family` | Map through `xw_hazard`. "Man-made" goes to `audit.etl_rejects` (DQ7) |
| ICA `ORIGINAL LOSS VALUE` | `original_loss_value` | `fact_event_loss.original_loss_aud` | Strip `$`, commas and spaces, then cast. `-` becomes 0 (DQ3) |
| ICA `NORMALISED LOSS VALUE (2022)` | `normalised_loss_value_2022` | `fact_event_loss.normalised_loss_2022_aud` | Same rule as above |
| ICA `TOTAL CLAIMS RECEIVED` and the 9 claim-type columns | `total_claims_received`, etc. | `total_claims` and the claim-type count columns | Same cast rule. `-` becomes 0, blank becomes NULL |
| ICA `Financial Year` | not loaded | `dim_date.au_financial_year` | ICA names the year by its start year, so it is not used. `dim_date` uses the standard convention (see section 4) |
| DE `id` | `id` | `dim_event.de_source_id` | Pass through when matched |
| DE `title` | `title` | hazard group and title text (used for matching) | Split on ` - `. The first part maps through `xw_hazard` |
| DE `startDate` | `startdate` | used for matching | Try month-first, then day-first (DQ5) |
| DE `Deaths`, `Injuries`, `Evacuated`, `Home(s) destroyed` | same, cleaned | `fact_event_loss.deaths`, `injuries`, `evacuated`, `homes_destroyed` | Cast to INT. Only for matched events |
| DE `Insured Cost` | not loaded | none | Unstated-year dollars, a different measure from ICA loss (see section 4) |
| ABS population: `Code`, `Label`, `Year`, `Estimated resident population (no.)` | `code`, `label`, `year`, `estimated_resident_population_no` | `fact_state_year.population` | Keep rows where `Code` matches `^[1-8]$`. Map `Label` through `xw_state`. `-` means no value (not 0) |
| ABS income: `Code`, `Label`, `Year`, median income column | same pattern | `fact_state_year.median_income_aud` | Same filter. `-` means no value. Column confirmed as `Median total income (excl. Government pensions and allowances) ($)` |
| ABS `Year` | `year` | `fact_state_year.date_key` | 30 June of the year, plus `population_year_used` and `income_year_used` with imputed flags (DQ6) |

## 3. Matching logic

There is no shared key across the sources, so each join uses attributes.

- **ICA to disaster events** (`xw_event_match`): same `state_code`, same `hazard_group`, and start dates within 7 days. If several candidates qualify, take the closest date, then the smallest Levenshtein name distance. Each disaster-events row may match at most one ICA event, checked with `GROUP BY de_row HAVING count(*) > 1`.
- **ICA to ABS:** state code and year, through `dim_location` and `dim_date`. Nearest-year imputation fills gaps.
- **Expected match rate is low.** Disaster events stops in 2014, and only 44 of its events start in 2011 or later.

## 4. Semantic inconsistencies

1. **Financial year.** ICA labels a March 2026 event FY25. The standard Australian financial year (July to June, named by the ending year) would call it FY26. `dim_date` uses the standard convention, so the report notes the one-year offset against ICA's labels.
2. **Insured cost.** ICA gives nominal and 2022-normalised loss. Disaster events has its own `Insured Cost` in unstated-year dollars. They are related but different measures, so only ICA figures load into the fact table.
3. **State.** ICA uses state codes and sub-state regions (FNQ, SEQ, SWQ), plus blanks, mixed case (VIC, Vic) and lists ("NSW, QLD", "NSW and SEQ"). ABS uses full state names. Everything is conformed through `xw_state`.
4. **Hazard type.** ICA uses a single `Type` field ("Storm"). Disaster events embeds the type in the title ("Severe Storm - Brisbane"). Both map to one hazard group through `xw_hazard`.
## 5. Profiling evidence (before-state)

| Finding | Evidence |
|---|---|
| 497 of 747 ICA rows are "Undeclared" (251 distinct CAT names) | `evidence/screenshots/p2_events_vs_rows.png` |
| 514 blank states, FNQ 167, SEQ 40, plus lists and mixed case | `evidence/screenshots/p2_state_values.png` |
| 27 dates parse into 2027 or later | `evidence/screenshots/p2_two_digit_years.png` |
| 2 dash placeholders, 699 blank claim counts | `evidence/screenshots/p2_placeholders.png` |
| Disaster events dates: 533 month-first, 140 day-first | `evidence/screenshots/p2_date_formats.png` |
| ABS population has values only for 2020–2025 (8 states each year) | `evidence/screenshots/p2_abs_population_years.png` |
| ABS median income has values only for 2019–2023 (8 states each year) | `evidence/screenshots/p2_abs_income_years.png` |