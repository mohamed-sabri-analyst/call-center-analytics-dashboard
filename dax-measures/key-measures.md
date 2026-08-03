# Key DAX Measures — Call Center Analytics Dashboard

This file documents the core DAX measures powering the dashboard, extracted directly from the Power BI data model. The full model contains 69 measures across 10 tables (`FactCalls`, `DimAgent`, `DimDate`, `DimQueue`, `Schedule_Table`, `Forecast`, `Targets`); the measures below are the ones driving the main KPI cards and visuals on each report page.

> Note: two measures shown below (`FCR Actual`) were corrected from the original build — see the note under Customer Experience.

---

## Executive KPIs

### Total Calls
```DAX
Total Calls = COUNTROWS(FactCalls)
```

### Total Answered
```DAX
Total Answered = 
CALCULATE(COUNTROWS(FactCalls), FactCalls[Answered] = 1)
```

### Total Abandoned
```DAX
Total Abandoned = 
CALCULATE(COUNTROWS(FactCalls), FactCalls[Abandoned] = 1)
```

### Abandon Rate %
```DAX
Abandon Rate % = 
DIVIDE([Total Abandoned], [Total Calls], 0)
```

### Service Level % (calls answered within 20 seconds)
```DAX
Service Level % = 
VAR AnsweredWithinSL = 
    CALCULATE(
        SUM(FactCalls[Answered]),
        FILTER(FactCalls, FactCalls[ASA] <= 20)
    )
VAR TotalCalls = [Total Calls]
RETURN
    DIVIDE(AnsweredWithinSL, TotalCalls, 0)
```

---

## Handle Time & Speed of Answer

### Average Handle Time (Minutes)
```DAX
Avg AHT (Minutes) = 
DIVIDE([Avg AHT (Seconds)], 60, 0)
```

```DAX
Avg AHT (Seconds) = 
CALCULATE(
    AVERAGE(FactCalls[AHT]),
    FactCalls[Answered] = 1
)
```

### Average Speed of Answer (ASA)
```DAX
Avg ASA (Sec) = 
CALCULATE(AVERAGE(FactCalls[ASA]), FactCalls[Answered] = 1)
```

### AHT vs Target Status
```DAX
AHT Status = 
IF(
    [AHT vs Target] <= 0,
    "✅ Within Target",
    IF([AHT vs Target] <= 30, "⚠️ Slightly Over", "❌ Over Target")
)
```

---

## Customer Experience

### First Call Resolution (FCR) Rate
```DAX
FCR Calls = 
CALCULATE(COUNTROWS(FactCalls), FactCalls[FCR] = 1, FactCalls[Answered] = 1)
```

```DAX
FCR Rate % = 
DIVIDE([FCR Calls], [Total Answered], 0)
```

**FCR Actual** (used in the Executive Insights decomposition tree):
```DAX
FCR Actual = [FCR Rate %]
```
> **Correction note:** the original measure was `DIVIDE([FCR Rate %], 100)`, which divided an already-fractional value by 100 again, producing an incorrect ~1% result instead of the true ~68%. Since `FCR Rate %` is already a ratio (e.g. 0.68), it should be referenced directly.

### Average CSAT Score
```DAX
Avg CSAT Score = 
CALCULATE(
    AVERAGE(FactCalls[CSAT]),
    FactCalls[Answered] = 1,
    NOT ISBLANK(FactCalls[CSAT])
)
```

### CSAT Distribution
```DAX
CSAT 5 Stars % = 
DIVIDE(
    CALCULATE(COUNTROWS(FactCalls), FactCalls[CSAT] = 5),
    CALCULATE(COUNTROWS(FactCalls), NOT ISBLANK(FactCalls[CSAT])),
    0
) * 100
```

---

## Workforce Management

### Schedule Adherence %
```DAX
Avg Adherence % = 
AVERAGE(Schedule_Table[Adherence_Pct])
```

### Shrinkage %
```DAX
Avg Shrinkage % = 
VAR NonProductiveHours = 
    SUM(Schedule_Table[Leave_Hours]) + SUM(Schedule_Table[Overtime_Hours])
VAR ScheduledHours = 
    SUM(Schedule_Table[Scheduled_Hours])
RETURN
    DIVIDE(NonProductiveHours, ScheduledHours, 0)
```

### Occupancy %
```DAX
Occupancy % = 
DIVIDE(
    CALCULATE(SUM(FactCalls[TalkTime]) + SUM(FactCalls[WrapTime])),
    CALCULATE(SUM(Schedule_Table[Actual_Hours])) * 3600,
    0
) * 100
```

### Utilization %
```DAX
Utilization % = 
DIVIDE([Total Actual Hours], [Total Scheduled Hours], 0) * 100
```

---

## Agent Ranking

### Agent Rank by CSAT
```DAX
Agent Rank by CSAT = 
RANKX(
    ALLSELECTED(DimAgent[AgentName]), 
    [Avg CSAT Score], 
    , 
    DESC, 
    Dense
)
```

### Top Performer Flag
```DAX
Top Performer Flag = 
IF([Agent Rank by CSAT] <= 10, "⭐ Top 10", "—")
```

---

## Forecast vs. Actual

### Forecast Accuracy
```DAX
Avg Forecast Accuracy % = 
AVERAGE(Forecast[Forecast_Accuracy])
```

### Forecast Accuracy Status
```DAX
Forecast Accuracy Status = 
IF(
    [Avg Forecast Accuracy %] >= 90,
    "Excellent",
    IF([Avg Forecast Accuracy %] >= 80, "Acceptable", "Poor")
)
```

---

## Time Intelligence

### Calls Month-over-Month Change %
```DAX
Calls This Month = 
CALCULATE([Total Calls], DATESMTD(DimDate[Date]))
```

```DAX
Calls MoM Change % = 
DIVIDE(
    [Calls This Month] - [Calls Last Month],
    [Calls Last Month], 0
) * 100
```

### Calls Year-over-Year Change %
```DAX
Calls YoY Change % = 
VAR CurrYearCalls = CALCULATE([Total Calls], DATESYTD(DimDate[Date]))
VAR PriorYearCalls = CALCULATE([Total Calls], DATEADD(DATESYTD(DimDate[Date]), -1, YEAR))
RETURN 
    DIVIDE(CurrYearCalls - PriorYearCalls, PriorYearCalls, 0) * 100
```

---

## Dynamic Labels (for report titles/tooltips)

```DAX
Date Range Label = 
"From " & FORMAT(MIN(DimDate[Date]), "MMM YYYY") &
" to " & FORMAT(MAX(DimDate[Date]), "MMM YYYY")
```

```DAX
Selected Queue Label = 
IF(
    ISFILTERED(DimQueue[QueueName]),
    SELECTEDVALUE(DimQueue[QueueName], "All Queues"),
    "All Queues"
)
```

---

**Author:** Mohamed Sabri Al-Deip
