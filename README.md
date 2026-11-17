<p align="center">
  <img src="assets/HireInsight_logo_horizontal.png" alt="HireInsight logo" width="420">
</p>

# Recruitment Dashboard (Power BI)

An interactive Power BI report on recruitment cost, speed and quality of hire, built for the **ZoomCharts Power BI Challenge, November 2026**.

The main question behind it: **which hiring channels actually deliver successful hires, and what do they really cost?**

The report uses **three ZoomCharts Drill Down PRO visuals** (Donut, Scatter and Combo Bar), one on each page, so viewers can drill from channel group to channel to provider.

---

## Screenshots

| Overview | Channels and funnel | Quality of hire |
|---|---|---|
| ![Overview](screenshots/page1_overview.png) | ![Channels and funnel](screenshots/page2_channels_funnel.png) | ![Quality of hire](screenshots/page3_quality.png) |

> Add your own screenshots to the `screenshots/` folder using the file names above.

---

## Key findings

Across 2024 to September 2026 the business made **4,048 hires** from **141,271 applicants** in **609 campaigns**, for a total spend of **€9.58M** (about **€2,367 per hire**, **55.5 days** to fill).

- **Cheap is not always efficient.** Social media costs €988 per hire, but only 40% of its hires are successful and 31% leave within 90 days. That lifts its cost per successful hire to €2,524, about 2.6 times the headline figure.
- **Referrals deliver the best quality.** Employee referrals have an 82% successful-hire rate, 3.5% early attrition, the fastest time to fill (41 days) and €1,638 per successful hire.
- **Agencies are expensive.** They take 59% of total spend for 12% of hires, at €11,339 per hire.
- **Job boards bring volume.** 27% of all hires, but with 17% early attrition.
- **Trend.** Cost per hire fell from €2,475 in 2024 to €2,199 in 2026, and hiring volume in January to September 2026 is up 28.7% on the same period of 2025.

> These are descriptive comparisons. Role mix differs by channel and the data contains no budgets or vacancy targets, so none of this is a forecast or a guaranteed saving.

---

## Report pages

1. **Overview**: KPI cards (Hires, Spend, Cost per Hire, Time to Fill) with current year, previous year and change, hires and cost trend by month, a location table, **spend by channel in a ZoomCharts Drill Down Donut PRO**, top 5 channel growth, and a slicer-driven Top/Bottom ranking.
2. **Channels and funnel**: conversion rates, a six-stage recruitment funnel, a channel and provider matrix, a **cost-versus-speed ZoomCharts Drill Down Scatter PRO**, cost mix by channel and time to fill by seniority.
3. **Quality of hire**: 90-day probation pass rate, 180-day retention, successful hire rate, performance score, cost per hire against cost per successful hire, a department-by-seniority heatmap, **early attrition in a ZoomCharts Drill Down Combo Bar PRO**, and a quarterly trend.

---

## ZoomCharts visuals

| Page | Visual | What viewers can do |
|---|---|---|
| Overview | Drill Down Donut PRO | Click a slice to drill from channel group to channel to provider, showing recruitment spend |
| Channels and funnel | Drill Down Scatter PRO | Compare cost per hire against time to fill by channel, then drill into providers |
| Quality of hire | Drill Down Combo Bar PRO | See 90-day early attrition by channel, then drill into providers |

All three use text columns from the channel dimension as their drill levels, and the same measures as the rest of the report, so they respond to the Year slicer like every other visual.

---

## Data model

A star schema with one fact table and three dimensions, plus a date dimension. All relationships are one-to-many, single direction.

```
DimDate ─────┐
DimRole ─────┤
DimChannel ──┼──► FactRecruitment
DimLocation ─┘
```

| Table | Rows | Purpose |
|---|---|---|
| `FactRecruitment` | 2,361 | Campaign-level funnel, cost, speed and quality-of-hire data |
| `DimDate` | 1,096 | Calendar for 2024 to 2026, marked as the date table |
| `DimChannel` | 8 | Channel, channel group, provider and colour |
| `DimLocation` | 4 | Region, country and city |
| `DimRole` | 36 | Department, role family, seniority and role title |
| `Top Bottom`, `Measure By`, `Top N` | small | Disconnected parameter tables for the ranking visual |
| `Funnel Stage` | 6 | Disconnected table that drives the funnel chart |
| `_Measures` | n/a | Holds all DAX measures, organised in display folders |

Model settings: automatic date/time is **off**, surrogate keys are hidden, and Month, YearMonth and Seniority use sort-by columns.

---

## DAX measures

The model has **52 measures** in the `_Measures` table, grouped into display folders:

| Folder | Measures | Examples |
|---|---|---|
| 1 Base | 10 | Hires, Applicants, Recruitment Spend, Cost per Hire, Avg Time to Fill |
| 2 Time Intelligence | 13 | Hires CY, Hires PY, Hires YoY %, plus the same for Spend, Cost per Hire and Time to Fill |
| 3 Ranking | 3 | Selected Metric, Rank Value, Ranked Metric |
| 4 Growth and Share | 2 | Top 5 Channel Hires Growth %, Spend Share % |
| 5 Funnel | 7 | Qualified Applicants, Interviewed Candidates, Accepted Offers, Funnel Value, conversion rates |
| 6 Quality of Hire | 6 | 90-Day Probation Pass Rate, 180-Day Retention, Successful Hire Rate, Cost per Successful Hire |
| 7 Formatting | 11 | Colour, label and period-caption measures for conditional formatting |

### Design decisions worth noting

**Like-for-like year-over-year on a partial year.** The data ends on 30 September 2026, so a naive comparison would set nine months against twelve. The current-year measure is capped at the latest date in the data, and the prior-year measure is cut off at the same point a year earlier.

```dax
Latest Data Date =
CALCULATE ( MAX ( FactRecruitment[CloseDate] ), ALL ( FactRecruitment ) )

Hires CY =
VAR _Latest = [Latest Data Date]
RETURN
    CALCULATE ( [Hires], KEEPFILTERS ( DimDate[Date] <= _Latest ) )

Hires PY =
VAR _Cut = EDATE ( [Latest Data Date], -12 )
RETURN
    CALCULATE (
        [Hires],
        SAMEPERIODLASTYEAR ( DimDate[Date] ),
        DimDate[Date] <= _Cut
    )
```

`KEEPFILTERS` matters here. Once `DimDate` is marked as a date table, a plain date filter inside `CALCULATE` clears the Year slicer.

**Quality metrics use the right denominators.** Retention and probation rates divide by the eligible hires only, so recent campaigns that cannot be assessed yet do not drag the rates down.

```dax
180-Day Retention =
DIVIDE ( SUM ( FactRecruitment[Retained180] ), SUM ( FactRecruitment[Eligible180Hires] ) )

Cost per Successful Hire =
DIVIDE (
    CALCULATE ( [Recruitment Spend], FactRecruitment[Mature180] = 1 ),
    SUM ( FactRecruitment[Successful180] )
)
```

**Slicer-driven Top/Bottom ranking.** Three disconnected tables let the viewer choose Top or Bottom, the metric and N.

```dax
Rank Value =
IF (
    SELECTEDVALUE ( 'Top Bottom'[Value], "Top" ) = "Top",
    RANKX ( ALLSELECTED ( DimRole[RoleTitle] ), [Selected Metric], , DESC, DENSE ),
    RANKX ( ALLSELECTED ( DimRole[RoleTitle] ), [Selected Metric], , ASC, DENSE )
)

Ranked Metric =
VAR _N = SELECTEDVALUE ( 'Top N'[Top N Value], 5 )
RETURN IF ( [Rank Value] <= _N, [Selected Metric] )
```

---

## How to use this report

1. Download `Recruitment_Dashboard.pbix`.
2. Open it in **Power BI Desktop** (any recent version).
3. Use the **Year** slicer on the left, which is synced across all pages. Year-over-year figures need exactly one year selected.
4. Use the page tabs at the top to move between pages, and **Reset** to clear filters.

---

## Repository structure

```
.
├── Recruitment_Dashboard.pbix
├── README.md
├── assets/
│   ├── HireInsight_logo_horizontal.png
│   └── HireInsight_icon.png
└── screenshots/
    ├── page1_overview.png
    ├── page2_channels_funnel.png
    └── page3_quality.png
```

---

## Limitations

- Spend is recruitment acquisition cost, not budget variance. The data has no budgets or vacancy targets.
- Channel comparisons are descriptive, because role mix varies by channel.
- Quality-of-hire metrics only include campaigns old enough to be assessed, so recent months are blank.
- 2026 covers January to September only.

---

## Tools

Power BI Desktop, DAX, star-schema modelling, ZoomCharts Drill Down PRO visuals (Donut, Scatter, Combo Bar).

## Credits

Dataset and challenge by **ZoomCharts**, November 2026. Report design, model and DAX by [Your Name].

## License

[Choose a license, for example MIT.]
