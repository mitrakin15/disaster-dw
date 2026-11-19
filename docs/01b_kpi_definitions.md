#**KPIS AND THEIR DEFINITIONS:**
| KPI | Formula | Additive? | Why it matters |
|-----|---------|-----------|----------------|
| Total normalised insured loss | SUM(normalised_loss_2022_aud) | Yes | Compares events across decades in 2022 dollars |
| Total nominal insured loss | SUM(original_loss_aud) | Yes | What was actually paid at the time |
| Event count | COUNT(event_key) | Yes | Frequency of catastrophes |
| Average loss per event | total loss ÷ event count | No | Severity trend (business question 2) |
| Loss per resident | total loss ÷ state population for that year | No | Fair comparison across states of different size (business questions 3 and 5) |
| Average cost per claim | total loss ÷ total claims | No | Claims budgeting. Only about 30% of events have claim counts, so report its coverage. |
| Loss concentration | share of loss from the top 10% of events | No | Reinsurance is bought for tail events |