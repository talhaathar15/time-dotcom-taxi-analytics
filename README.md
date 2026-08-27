# Time DotCom — Chicago Taxi Analytics
This README documents my process, assumptions, and findings for each part of the assessment, written in the order I actually worked through them.
**Stack:** BigQuery (warehouse) - Dataform (transformation) - Looker Studio (visualization)
**Dataset:** `bigquery-public-data.chicago_taxi_trips.taxi_trips`
**Dashboard:** https://datastudio.google.com/reporting/842e055b-dc4f-4dcd-8282-f0bd70e8d990
**GitHub repo:** https://github.com/talhaathar15/time-dotcom-taxi-analytics


## Setup
- BigQuery project: `time-dotcom-analytics`
- Dataform repository: `taxi-analytics-dataform`, region `us-central1`
- All models write to the `taxi_analytics` dataset, `US` multi-region (matches the location of `bigquery-public-data`)
- Dataform config (`workflow_settings.yaml`) explicitly sets `defaultLocation: US`. I initially left this unset and hit a "table not found in location US" error until I added it explicitly.
- One thing worth flagging in case it comes up: the inline "Run" button in the Dataform code editor only previews a query and does *not* materialize a real table. You have to use "Start execution" for that. I lost some time on this early on because the preview looked identical to a successful run, and only caught it when `INFORMATION_SCHEMA` showed the table didn't actually exist.


### Data model overview
stg_taxi_trips (staging)
├── tip_earners (mart — Q1)
├── int_taxi_shifts (intermediate)
│ ├── overworkers (mart — Q2)
│ └── tip_efficiency (mart — Also references tip_earners)
└── holiday_impact (mart — Q3)



## Question 1 — Top 100 Tip Earners

### Understanding the date range

The dataset's most recent trip timestamp is `2023-12-31 23:45:00 UTC` — it doesn't extend to the present. So "last 3 months" is the 3 calendar months ending at the dataset's own max date: **October 1 – December 31, 2023**, not relative to today.

### Data quality investigation

Before writing any cleaning logic, I looked at the raw data for this window and found some clearly broken rows - e.g. a $9,666.66 fare for a 3-minute, 0-mile trip. Checking the top 50 highest-fare trips manually, almost all of them showed this same pattern: near-zero distance and duration paired with a huge fare. One outlier had 1,132 miles logged in 4.4 hours, which works out to about 257 mph and it's also clearly wrong.

I didn't want to just pick an arbitrary dollar cutoff, so I looked at the actual percentile distribution of implied speed (miles ÷ hours) and fare-per-minute across the window. Speed was well-behaved all the way to the 99th percentile (~51 mph). But the Fare-per-minute was not and it jumped from $3.25 at the 95th percentile to over $200 at the 99th. Narrowing in further, the real "elbow" where the data goes from normal to broken sits around the 97th percentile (~$10/minute), which also roughly matches Chicago's actual published taxi fare structure (flag drop + per-mile + per-minute rates), where even a fast trip rarely exceeds $8–10/minute in genuine fare.

So the final cleaning rule is to exclude any trip where implied speed exceeds 60 mph, or fare-per-minute exceeds $10. This is about 3.16% of trips in the window (49,269 out of around 1.56M), but it's not negligible, so I wanted the rule to be evidence-based rather than just to guess.

I also checked whether this bad data was concentrated in one company, thinking maybe a meter issue specific to one fleet but it wasn't. The rate ranged from 1.7% to 5.74% across companies with no single outlier, so I treated it as a general data quality issue rather than flagging one company specifically.

One more thing I caught while checking by company breakdowns is that the same company was appearing twice under slightly different names - "Taxicab Insurance Agency Llc" and "Taxicab Insurance Agency, LLC" (has comma difference). Fixed by stripping commas and standardizing case/whitespace in the staging model.

### Other exclusions

- 27 trips had a missing `trip_end_timestamp` so I excluded those, since they can't be used for duration-based calculations anyway (which is also relevant for Question 2) and the row count is small enough.
- Cash tips are not reliably recorded in this dataset but I noted directly in Google's own schema description. This means the "top tip earners" ranking really reflects *recorded* tips with mostly card payments and not the total real-world tips.

### Data model

- **`stg_taxi_trips`** — cleaned staging table: filters to the Oct–Dec 2023 window, drops rows with missing end timestamps or non-positive duration, excludes implausible fare/speed outliers (rule above), and normalizes company names (uppercase, trimmed, commas stripped). 1,504,164 rows.
- **`tip_earners`** — aggregates `stg_taxi_trips` by `taxi_id`, ranks by total recorded tips, top 100. Also includes total trips, total fare, total trip revenue, and average tip per trip as extra context beyond just the raw total.

### What I found

Six of the top 10 taxis by total tips belong to just three companies (Sun Taxi, Taxicab Insurance Agency LLC, Taxi Affiliation Services). This is worth flagging as a caveat rather than a clean finding and the total tips rewards trip volume as much as anything else, so this concentration likely reflects fleet size more than individual driver skill. 



## Question 2 — Top 100 Overworkers

### Breaking down what's actually being asked

The question packs two separate conditions into one sentence so taxis that (1) don't take at least 8 hours break between shifts, and (2) regularly have a long shift. Neither "shift" nor "long" nor "regularly" is a field in the data. A shift is something I had to derive from the pattern of a taxi's trips over time and "long" and "regularly" both needed a definition I could actually defend without guessing.

### Researching what a real taxi shift looks like, before touching data

Before deciding on any numeric threshold, I looked into how Chicago taxi shifts actually work in practice:

- A 2009 comprehensive study of the Chicago taxicab industry found the *average* shift is over 13 hours.
- Chicago's own taxi ordinance sets a legal maximum of 12 consecutive hours behind the wheel.
- Multiple news pieces (WTTW, Sun-Times, NPR) quote real Chicago drivers describing 12–17 hour days as normal, especially given competition from Uber/Lyft eating into per-trip earnings.

This gave me an externally-grounded number to anchor "long shift" to 12 hours — consider the city's own legal driving limit without just an arbitrary round number.

### Detecting shifts from raw trip data

The data gives individual trips with start/end timestamps. I used a window function (`LAG()`) to compare each trip to the previous one for the same taxi, measuring the gap since the last trip ended. As per the question's own definition, a "break" is at least 8 hours, so if the gap since the last trip is ≥480 minutes, the next trip starts a new shift.

The grouping mechanism is that flag each "new shift start" as 1 and everything else as 0, then take a running `SUM()` of that flag per taxi ordered by time. A cumulative sum of 0/1 flags naturally creates a unique, incrementing shift ID.

Before trusting this at scale, I tested it on one real taxi (my top tip-earner from Q1). The gap distribution showed a completely clean separation within shift gaps topped out around 150 minutes and between shift gaps started at 555 minutes — nothing falls in between. That validated 8 hours used as a genuinely clean cutoff without any assumption. Detected shifts for this taxi mostly landed in the 8–16 hour range and the matching the research closely.

### The multi-day "mega-shift" problem

Running this logic across every taxi, the average shift came out to around 8.5 hours, but the maximum was **745 hours** is about 31 days continuously "on shift." About 14.8% of all detected shifts spanned multiple calendar days.

Looking closer, the same `taxi_id` appeared repeatedly across multiple multi-week "mega-shifts," and trip count scaled almost exactly with duration (718 trips over 745 hours — roughly one trip per hour, nonstop for a month). That's not a human pattern but a vehicle in near continuous operation.

The explanation is Chicago taxi medallions are commonly leased across multiple drivers who split day/night shifts on the same physical car. `taxi_id` represents the vehicle/medallion without an individual person — so a handoff between two drivers under 8 hours apart gets incorrectly stitched into one "shift" by the logic I used.

There's no driver ID in this dataset, so I have no direct way to detect the actual handoff point. Rather than guess I added an `is_plausible_single_driver_shift` flag so any shift longer than 16 hours comfortably above Chicago's 12-hour legal limit, with buffer is flagged as not plausible for one driver and excluded from the "long shift" judgment specifically. The underlying "went 8+ hours without a trip" signal is kept regardless of shift length, since that's still a legitimate fact about vehicle utilization independent of driver count.

After this fix, only 3.13% of shifts were flagged implausible. Average plausible shift length came out to 7.73 hours across all taxis and shifts which is lower than the 13-hour research figure because this average includes short/partial shifts too and not just full working days).

### Defining "regularly," using the actual distribution

I looked at what percentage of each taxi's shifts were "long" (>12h), across taxis with at least 10 shifts to avoid a taxi with only 2–3 shifts looking artificially extreme by chance:

| Percentile | % of shifts that are long |
|---|---|
| Median | 6.82% |
| 75th | 23.08% |
| 90th | 41.18% |
| 95th | 52.00% |

There's a clear separation between "typical" and an occasional long day and this is basically how this taxi operates. I set "regularly" at ≥40% of a taxi's shifts being long, aligning with the 90th percentile — a genuinely distinct group with no arbitrary line on curve.

### Data model

- **`int_taxi_shifts`** — intermediate model, one row per detected shift per taxi. Includes shift start/end, duration, trip count, calendar days spanned, and the `is_plausible_single_driver_shift` flag.
- **`overworkers`** — Used as final mart. Filters to plausible single-driver shifts, requires ≥10 shifts for reliability, keeps taxis with ≥40% long shifts, ranks top 100 by total hours spent in long shifts to rewards genuine volume of overwork without just barely crossing the threshold.

### What I found

The #1 overworker taxi had 79 of 80 shifts (98.75%) exceeding 12 hours, averaging 14 hours per shift. The top 10 showed internally consistent numbers and nothing above the 16-hour cap sneaking through and a reasonable spread across companies.

**Limitation worth stating plainly:** because `taxi_id` represents a vehicle, not a confirmed single driver and shorter multi-driver handoffs under 16 hours can't be fully ruled out, "overworker" here describes vehicle usage patterns consistent with one driver working excessive hours and not a verified proof of one person's schedule. I think it's still a fair and useful proxy given what the dataset provides..



## Question 3 — Did public holidays impact trip volume?

It depends heavily on the holiday.

I checked the four federal holidays falling in the Oct–Dec 2023 window: Columbus Day, Veterans Day (both the actual date, a Saturday and Friday), Thanksgiving and Christmas.

To measure impact fairly, I compared each holiday's trip count against the average for that same day of the week (e.g., Christmas Monday vs. other Mondays), rather than the overall daily average and taxi volume already varies a lot by day of week regardless of holidays, so a raw comparison would conflate the two effects.

| Holiday | Day | Trip count | Normal for that weekday | % difference |
| Columbus Day (Oct 9) | Monday | 19,593 | 16,553 | initially +18.4% |
| Veterans Day observed (Nov 10) | Friday | 17,965 | 17,775 | +1.1% |
| Veterans Day actual (Nov 11) | Saturday | 12,687 | 12,979 | -2.2% |
| Thanksgiving (Nov 23) | Thursday | 6,800 | 18,751 | -63.7% |
| Christmas (Dec 25) | Monday | 4,885 | 16,553 | -70.5% |

**Note on Columbus Day:** my first pass showed +18.4% against the full-quarter Monday average but checking it against only its immediate neighboring Mondays (Oct 2, 16, 23) showed it was actually unremarkable, falls right in the middle of the range. The earlier number was an artifact of overall trip volume trending downward from October into December. 

**My take:** Thanksgiving and Christmas are the two biggest "stay home with family" holidays in the US and the data shows it clearly, both see 64–70% drops versus a typical day of that weekday. Columbus Day and Veterans Day are federal government holidays that most private employers and the general public don't meaningfully observe, so daily routines and taxi demand carry on largely as normal.

**Data note:** December 31 the dataset's final day shows only 71 trips, versus 7,000–15,000+ for every other Sunday in the window. This is almost certainly an incomplete-data artifact from the export cutoff, not a real pattern, and was excluded from this analysis.

### Data model

**`holiday_impact`** Shows one row per day in the window, with day-of-week baseline comparison and the 5 holiday dates flagged by name.



## Bonus Insight 1 — "Top tip earners" rewards hours worked

Ranking taxis by total tips (Question 1) mostly reflects how many hours a taxi was in operation, not how well it performs per hour. I built an alternative ranking using tips-per-hour (total tips ÷ total hours in plausible single-driver shifts, minimum 50 hours worked for a stable rate) and the results barely overlap with the original ranking — only 19 of the top 100 by efficiency also appear in the top 100 by total tips.

More importantly, I cross-referenced both rankings against the Question 2 overworkers list. 27 of the original top-100 total-tip earners are also flagged as overworkers which means over a quarter of what looks like a "top performer" list is really just a "works excessive hours" list. By contrast, zero of the top 100 by tips-per-hour appear on the overworkers list at all.

**Business value:** if this data is ever used to identify or reward top-performing drivers or companies, using total tips as the metric would end up implicitly rewarding overwork rather than skill or service quality. Tips-per-hour is a fairer, safer metric for that purpose and this analysis shows it identifies a meaningfully different, non-overlapping group of taxis.

### Data model

**`tip_efficiency`** Shows tips-per-hour per taxi (min. 50 hours), with boolean flags showing overlap against both `tip_earners` and `overworkers`.

---

## Bonus Insight 2 — Data quality issues carry real operational risk, not just analytical noise

During data cleaning for Question 1, I found that 3.16% of trips in the Oct–Dec 2023 window had physically implausible values with near zero-distance trips with fares in the thousands of dollars and one trip implying a 257 mph average speed. This wasn't concentrated in one company (rates ranged 1.7%–5.74% across companies with no single outlier), suggesting a general reporting/meter issue across the fleet rather than one bad actor. I also found the same company appearing under inconsistent name formatting (e.g., "Taxicab Insurance Agency Llc" vs "Taxicab Insurance Agency, LLC"), which would silently split one company's totals across two rows in any report that groups by company name without normalizing it first.

**Business value:** these aren't just issues that needed handling for this analysis, if this same raw data feeds into billing, driver payouts or regulatory reporting elsewhere, unnoticed fare/meter errors and duplicate company entities could cause real financial discrepancies or inaccurate compliance reporting. I'd recommend a recurring, automated data quality check exactly the kind of check now built into the `stg_taxi_trips` model) rather than relying on manual review before the data reaches any downstream system.



## Dashboard structure (Looker Studio)

- **Q1: Tip Earners** — Top 10 taxis by total tips; Total tips by company
- **Q2: Overworkers** — Top 10 by total long-shift hours; Top 10 by % of shifts that are long
- **Q3: Holiday Impact** — Daily trip volume across the quarter, with labeled holidays.
- **Tip Efficiency** — Top 10 by tips-per-hour

## A note on how I worked through this

I want to be upfront that my process for several decisions, the fare/speed threshold, the "regularly" definition, the multi-day shift cap started with a reasonable first guess, which I then checked against the actual data distribution and revised where the evidence didn't support it, rather than getting it exactly right on the first try. 
