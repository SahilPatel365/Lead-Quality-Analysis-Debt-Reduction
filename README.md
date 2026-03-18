# Lead Quality Analysis: Debt Reduction Case Study

**Author:** Sahil Patel
**Date:** March 2026
**Tools:** R, R Markdown, ggplot2, dplyr, knitr
**Output:** Business report (Word/PDF) and full technical analysis (PDF)

---

## What This Project Is About

A lead generation publisher sells consumer leads to a debt reduction advertiser.
The advertiser pays $30 for every lead delivered, regardless of whether that lead
ever becomes a paying customer. The contract includes a quality target: at least
9.6% of leads delivered must convert to a closed customer. When that target is
met consistently, the advertiser increases the price per lead from $30 to $33.

At the time of this analysis, the publisher was hitting only 8.09%.

That 1.51 percentage point gap sounds small. It is not. On 2,857 leads over six
months, it represents 44 missing closed customers and $8,571 in unrealised revenue
every six months, or $17,142 per year, all from the same traffic volume with no
additional spend required.

This project answers one question: **why is quality below target, and what exactly
needs to change to fix it?**

---

## The Dataset

The raw dataset contained **3,021 leads** submitted between April 2 and September
30, 2009, covering 181 days. Each row represents one person who submitted a debt
reduction enquiry, either through an online form or via a call center agent.

The dataset recorded 24 variables per lead, including:

- Who brought the lead (traffic partner: Google, Yahoo, AdKnowledge, etc.)
- How much debt the consumer reported carrying (10 debt range categories)
- Which ad design (widget) the consumer saw and clicked on
- Whether the lead was followed up by a call center and what happened
- Address and phone verification scores (available for 46% of leads)
- Geographic location (US state), campaign names, ad keywords

The outcome variable is `CallStatus`, which records what happened after a lead
was submitted. Seven distinct outcomes were recorded: Closed, EP Confirmed, EP
Sent, EP Received, Contacted but Does Not Qualify, Invalid Profile, Unable to
Contact, and no record at all (71% of leads had no outcome recorded).

---

## How Quality Is Defined

Not every lead outcome counts the same way. For this analysis, outcomes were
grouped into three categories:

- **Good:** Leads that converted to a paying customer (Closed) or entered the
  advertiser's EP pipeline (EP Sent, EP Received, EP Confirmed). These represent
  genuine commercial value.
- **Bad:** Leads that were contacted but failed to qualify or could not be reached.
  These cost the advertiser $30 and returned nothing.
- **No Outcome:** Leads with no call center record at all. These were paid for at
  the full $30 CPL despite never being followed up. They make up 71% of all leads
  and are included in the denominator for all quality calculations because the
  publisher was paid for them regardless.

The **closed rate** (Closed leads only, divided by total unique leads) is the
primary metric used for business impact calculations. The current closed rate
is **8.09%**. The contract target is **9.6%**.

---

## The Nine-Phase Analysis

This project was structured as a rigorous nine-phase analytical process. Each
phase built directly on the previous one.

### Phase 1: Data Profiling

Before touching the data, the first step was to understand exactly what was in it.

The dataset has 3,021 rows and 24 columns. R correctly identified `LeadCreated`
as a datetime field. Two problems were flagged immediately: `AddressScore` and
`PhoneScore` were stored as character type (not numeric), and the `IP Address`
column was entirely empty with 3,021 NA values.

The `Partner` column returned 6 unique values instead of the expected 3. That
was a red flag. Investigation confirmed "google" (979 leads) and "Google" (639
leads) were being counted as two separate partners due to inconsistent
capitalisation. Combined, Google accounts for 53.6% of all leads. This issue,
if left uncorrected, would have split Google's data across two groups and made
every partner comparison in the project meaningless.

The `CallStatus` column had 70.8% missing values. Month-by-month breakdown
confirmed this was systemic across all six months, ranging from 62.7% in June
to 80.0% in May. This is not a technical error. These are leads that were simply
never followed up on, or whose outcomes were never recorded.

### Phase 2: Data Quality Assessment

Eight specific data quality issues were identified and documented before any
cleaning began. The full list:

| Column | Issue | Planned Fix |
|---|---|---|
| Partner | Capitalisation inconsistency: "google" vs "Google" | case_when() to standardise all values |
| AddressScore | String "NULL" instead of NA, stored as character | na_if() then as.numeric() |
| PhoneScore | String "NULL" instead of NA, stored as character | na_if() then as.numeric() |
| IP Address | 100% empty (3,021 NAs), no analytical value | Drop the column entirely |
| DebtLevel | Unordered character string, wrong alphabetic sort | Convert to ordered factor |
| DebtLevel | Overlapping ranges: "7500-10000" and "7500-15000" share the same lower bound | Retain both; collapse to "Under 15K" in Phase 4 |
| CallStatus | 70.8% missing, systemic across all months | Replace NA with "No Outcome"; recode into quality groups |
| Email | 133 duplicate email addresses (4.4% of leads) | Flag duplicates; exclude from quality rate calculations |

The most material issue for analytical accuracy was the Partner capitalisation
problem. Without fixing it, Google's 53.6% market share would have been
artificially split into two smaller groups, distorting every partner-level
comparison in the project.

### Phase 3: Data Cleaning

All eight issues were resolved in this phase. The approach was conservative:
no records were deleted. Duplicates were flagged with a new boolean column
(`is_duplicate_email`). The IP Address column was dropped. All other columns
were corrected in place.

After cleaning, the Partner column had five clean values: Google (1,618 leads,
53.6%), Yahoo (958, 31.7%), Call Center (271, 9.0%), AdKnowledge (171, 5.7%),
and Advertise.com (3, 0.1%).

The CallStatus column was recoded into a `QualityGroup` with three values:
Good (393 leads, 13.0%), Bad (488, 16.2%), and No Outcome (2,140, 70.8%).

Duplicate email flagging identified 164 rows as non-first occurrences of a
shared email address. The clean analysis base of **2,857 unique leads** was
established for all quality rate calculations going forward.

### Phase 4: Feature Engineering

Eleven new variables were created from the existing raw columns to make
segment-level analysis possible. The four most important ones:

- **QualityGroup:** Good, Bad, No Outcome (described above)
- **DebtBand:** The 10 raw debt ranges were collapsed into 8 ordered clean
  bands (Under 15K, 15K-20K, 20K-30K, 30K-50K, 50K-70K, 70K-90K, 90K-100K,
  Over 100K). The two overlapping lower bands were merged into "Under 15K."
- **widget_design:** The 14 raw widget name strings were simplified into 9
  readable design categories (CreditSolutions, BlueMeter, Head2, Head3,
  Standard, White, YellowArrow, YellowArrow Blue, YellowArrow Dark)
- **verify_score:** The average of `AddressScore` and `PhoneScore` where both
  values were available. This composite score was available for 46% of leads
  (1,315 out of 2,857) because PhoneScore was introduced partway through the
  data collection period.

Time-based features (month, week, day of week, hour) were also extracted from
`LeadCreated` to enable trend analysis.

### Phase 5: Exploratory Data Analysis

EDA was run across seven variables to identify which ones showed meaningful
differences in quality rate. Quality rate here is the percentage of leads in
a segment that fell into the Good category.

Key findings from EDA:

- **Overall baseline quality rate:** 13.0% (broad), 8.09% (closed only)
- **Partner:** AdKnowledge 18.7%, Google 11.2%, Yahoo 9.7%, Call Center 17.9%
- **Debt Band:** 70K-90K is highest at 20.2%, Over 100K is lowest at 6.7%
- **Widget Design:** YellowArrow 24.5%, Head2 16.3%, BlueMeter 14.4%, Head3 6.9%
- **Lead Source:** Call Center 17.9%, Online 12.8%
- **Day of Week:** No material pattern. Performance was flat across all days.
- **Ad Type:** No material difference between branded and generic campaigns.

The EDA made three variables stand out immediately: Partner, Debt Band, and
Widget Design. Every other variable either showed no meaningful pattern or
was not statistically actionable.

### Phase 6: Multi-Variable Analysis

Single-variable analysis shows averages. Multi-variable analysis shows where
those averages are hiding the real problem.

The most important cross-tab was **Partner x Debt Band**. When you break Google
down by debt range, the problem becomes very specific:

| Partner | Debt Band | Total | Quality Rate |
|---|---|---|---|
| Google | Over 100K | 107 | 4.7% |
| Google | 70K-90K | 76 | 13.2% |
| Google | 50K-70K | 135 | 12.6% |
| AdKnowledge | 50K-70K | 10 | 40.0% |

Google's Over 100K segment is the single worst-performing meaningful cell in
the entire dataset at 4.7% quality from 107 leads. This is 6.8 percentage
points below the 9.6% target. And it is not a small group; 107 leads is large
enough to drag the overall average down measurably.

The **Debt Band x Widget Design** cross-tab confirmed that widget performance
differences were consistent across debt levels, meaning the Head3 problem is
not a debt-level artefact. Head3 underperforms regardless of which consumers
see it.

### Phase 7: Statistical Hypothesis Testing

Six variables were formally tested for statistical association with lead quality.
Chi-square tests were used for categorical variables. Mann-Whitney U was used
for verify score because it is a continuous variable with a non-normal
distribution. Monte Carlo simulation was applied where cell counts were too
small for standard chi-square assumptions.

| Variable | Test | P-Value | Significant |
|---|---|---|---|
| Verify Score | Mann-Whitney U | 0.0003 | Yes *** |
| Debt Band | Chi-Square | 0.0010 | Yes *** |
| Partner | Chi-Square (Monte Carlo) | 0.0785 | No |
| Widget Design | Chi-Square | 0.0879 | No |
| Day of Week | Chi-Square | 0.7226 | No |
| Lead Source | Chi-Square | 0.9487 | No |
| Ad Type | Chi-Square | 1.0000 | No |

Verify score is the strongest statistical signal in the dataset (p = 0.0003).
Good leads had a median verify score of 5.0. Bad leads had a median of 4.0.
A threshold of 4.5 sits exactly between the two medians and maximises
separation between quality groups.

Debt Band is confirmed statistically significant (p = 0.001). The relationship
between debt level and lead quality is real, not random.

Partner and Widget Design are borderline. The practical effect sizes are real
and large (7.5 percentage points between AdKnowledge and Google, 9.4 percentage
points between Head3 and Head2), but the sample sizes, particularly for
AdKnowledge, were not large enough to reach the 0.05 threshold with certainty.
This does not make those findings less actionable. It means more volume would
be needed to reach statistical certainty.

Day of week, lead source, and ad type showed no statistically significant
relationship with quality at any threshold. No targeting changes are warranted
for these variables.

### Phase 8: Business Impact Modelling

The revenue model is straightforward. The publisher earns $30 per lead delivered,
regardless of quality. At 2,857 leads, that is $85,710 over six months.

If the closed rate reaches 9.6%, the CPL increases to $33. At 2,857 leads, that
is $94,281 over six months. The difference is $8,571 per six months, or
**$17,142 annualised**, with no change to lead volume and no additional traffic
spend.

To reach 9.6% from 8.09% on the same base of 2,857 leads, **44 additional
closed leads** are needed.

A scenario analysis was run to test whether simply cutting the worst-performing
segments would be enough to close the gap:

| Scenario | Total Leads | Closed Leads | Closed Rate |
|---|---|---|---|
| Baseline (all leads) | 2,857 | 231 | 8.09% |
| Remove Google Over 100K | 2,750 | 228 | 8.29% |
| Remove Head3 widget | 2,785 | 227 | 8.15% |
| Remove both | 2,679 | 224 | 8.36% |

None of the removal scenarios reach 9.6%. Even removing both the worst partner
segment and the worst widget raises the closed rate to only 8.36%. The gap
that remains, 1.24 percentage points, can only be closed through active mix
improvement, not passive elimination.

### Phase 9: Recommendations

Five recommendations were developed directly from the evidence. Each one has
a specific evidence base, a concrete action, an estimated impact in additional
Good leads per six-month period, and a realistic timeline.

**R1: Shift budget from Google Over 100K to AdKnowledge (70K-90K focus)**
Google's Over 100K segment delivers a 4.7% quality rate, the worst in the
dataset. AdKnowledge delivers 18.7% overall and consistently outperforms Google
across every debt band where both have volume. The action is to add negative
keyword exclusions in Google campaigns for debt-over-$100K queries and redirect
that budget to AdKnowledge campaigns targeting $50K-$90K consumers.
Estimated impact: 15 to 20 additional Good leads per six months.
Timeline: Week 1 to 2. Configuration only, no development required.

**R2: Remove Head3 widget from all campaigns**
Head3 delivers a 6.9% quality rate from 72 leads. Every other widget
outperforms it by at least 3.5 percentage points. Head2 runs at 16.3% and
BlueMeter at 14.4%. Replacing Head3 with either is a creative swap that takes
days, not weeks. Estimated impact: 5 additional Good leads per six months.
Timeline: Week 1 to 2. Creative swap only.

**R3: Deploy a verify score pre-filter at lead submission**
Mann-Whitney U test confirmed verify score is the strongest quality predictor
in the dataset (p = 0.0003). A minimum threshold of 4.5 sits between the Good
leads median (5.0) and Bad leads median (4.0), filtering out low-verification
leads before the advertiser is billed. This is the only recommendation that
requires engineering work (real-time API integration for AddressScore and
PhoneScore). Estimated impact: 8 to 12 additional Good leads per six months.
Timeline: Week 5 to 8.

**R4: Prioritise 70K-90K debt band in keyword targeting**
The 70K-90K band achieves a quality rate of 20.2%, the highest of any debt
segment. Chi-square confirmed the debt band relationship is statistically
significant (p = 0.001). The action is to increase keyword bids for debt
queries in the $50K-$90K range and reduce bids for Over $100K queries. Estimated
impact: 10 to 15 additional Good leads per six months. Timeline: Week 1 to 4.

**R5: Add pre-qualification to the online lead form**
Call center leads convert at 17.9% quality versus 12.8% for online leads,
a 5.1 percentage point gap. When comparing only leads that were actually
contacted, the rates are near-identical (47.1% vs 45.5%), which means the
advantage comes entirely from pre-screening, not from the channel itself. Call
center agents verbally qualify consumers before entering their details. The online
form currently has no equivalent filter. Adding a single debt range question to
the form and soft-gating submissions below $15K or above $150K replicates that
screening effect. Estimated impact: 6 to 10 additional Good leads per six months.
Timeline: Week 3 to 4.

---

## Results Summary

| Metric | Value |
|---|---|
| Raw leads | 3,021 |
| Clean unique leads (analysis base) | 2,857 |
| Current closed rate | 8.09% |
| Contract target closed rate | 9.6% |
| Closed leads gap to target | 44 leads |
| Current 6-month revenue | $85,710 |
| Target 6-month revenue | $94,281 |
| Annualised revenue uplift | $17,142 |
| Strongest statistical predictor | Verify Score (p = 0.0003) |
| Worst performing segment | Google + Over 100K (4.7% quality) |
| Best performing segment | AdKnowledge + 70K-90K (40.0% quality) |
| Combined estimated impact of all 5 recommendations | +44 to 62 Good leads per 6 months |

---

## Implementation Roadmap

| Phase | Weeks | Recommendations | Est. Lead Gain |
|---|---|---|---|
| Immediate | 1 to 2 | R1, R2 | +20 to 25 |
| Short-term | 3 to 4 | R4, R5 | +16 to 25 |
| Medium-term | 5 to 8 | R3 | +8 to 12 |
| **Total** | **8 weeks** | **All five active** | **+44 to 62** |

The minimum estimate of 44 additional Good leads is exactly the number
required to move from 8.09% to 9.6% on the same 2,857-lead base.

---

## Repository Structure

LeadQuality_CaseStudy/
│
├── data/
│ ├── Analyst_case_study_dataset_1.xlsx # Raw dataset (3,021 leads, 24 columns)
│ └── clean_data_final.xlsx # Cleaned and feature-engineered dataset
│
├── report/
│ ├── lead_quality_analysis.Rmd # Full technical analysis (9 phases, all R code)
│ ├── lead_quality_analysis.pdf # Rendered PDF of technical analysis
│ ├── Final_Report.Rmd # Business-facing report (R Markdown)
│ └── Final_Report.docx # Rendered Word document
│
├── charts/ # Selected chart PNGs (subset, full set generated on knit)
└── tables/ # Selected CSV tables (subset, full set generated on knit)


---

## How to Reproduce This Analysis

**Requirements:**
- R version 4.0 or above
- RStudio
- Packages: tidyverse, readxl, knitr, kableExtra, scales

**Steps:**

1. Clone or download this repository
2. Open `lead_quality_analysis.Rmd` in RStudio
3. Update the file paths at the top of the setup chunk to match your local directory
4. Click **Knit** to regenerate the full PDF
5. To reproduce the business report, open `Final_Report.Rmd` and knit to Word

All charts and tables are generated dynamically when the files are knitted.
The clean dataset (`clean_data_final.xlsx`) is required for both reports.

---

## Key Technical Skills Demonstrated

- Data profiling and systematic quality auditing across 8 issue categories
- Data cleaning in R using dplyr (case_when, na_if, factor ordering, duplicate flagging)
- Feature engineering: 11 new variables from raw columns including composite scoring
- Exploratory analysis across 7 dimensions with ggplot2 visualisations
- Multi-variable cross-tabulation analysis (Partner x Debt Band, Debt Band x Widget)
- Statistical hypothesis testing: Chi-Square (with Monte Carlo), Mann-Whitney U, Spearman Correlation
- Business impact modelling: revenue gap analysis, scenario testing, CPL uplift calculation
- Reproducible reporting in R Markdown exported to Word and PDF via Pandoc

---

## Contact

Sahil Patel
sahilgp365@gmail.com
github.com/SahilPatel365
