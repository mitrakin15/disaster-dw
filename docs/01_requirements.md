#**Decision Problem:**
- Which hazards and states should Kestrel prioritise for catastrophe reserves, reinsurance cover, premium reviews and claims staffing?

#**USERS:**
- The client is a fictional australlian general insurer named kestrel general insurance.
- the desicion makers who use the data warehouse are:

    1.Reinsurance and risk manager

    2.Head of Underwriting

    3.Claims operations manager

    4.Catastrophe pricing analyst

#**SCOPE:**
- In scope:

    - Australian natural-hazard events:
     cyclone, storm, flood, bushfire, hail, earthquake
    - Events from 2011 onward
    - Analysis at state level

- Out of scope:

    - Man-made events (e.g. COVID-19, Middle East conflict) and overseas events
    - Individual policies
    - Premium models
    - Region or postcode analysis

#**BUSINESS QUESTIONS AND USERS:**
| ID | Business question | User | Decision it supports |
|----|-------------------|------|----------------------|
| Q1 | Which hazard types and states produce the most normalised insured loss, and how concentrated is it? | Reinsurance and risk manager | Where to buy reinsurance cover |
| Q2 | Is the average loss per event rising over time, in nominal and in normalised dollars? | Head of Underwriting | Whether premiums need a trend loading |
| Q3 | Which states are hit most often, relative to their population? | Head of Underwriting | Where to review pricing and exposure |
| Q4 | Which months and seasons have the most events and claims? | Claims operations manager | When to schedule surge staff and reserves |
| Q5 | Do lower-income states have lower insured loss per person, suggesting underinsurance? | Catastrophe pricing analyst | Product and affordability strategy |

