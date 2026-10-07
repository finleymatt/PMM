# TOPS - Reporting Window Scheduler (Power Automate)

Scheduled cloud flow that opens and closes `ecm_reportingwindows` rows on the fixed calendar below, pre-creates
open Actual rows for quarterly measures, and closes Actual rows when a window ends. Build one copy per environment
(dev, test, prod) inside the TOPS solution. No HTTP trigger and no URL in the pages; the pages only read the
`ecm_reportingwindows` table.

The existing manual toggle on **Manage Reporting Windows** keeps working as an override. The scheduler only closes a
window during the three days after its scheduled end date, so a window opened by hand off-schedule stays open until
an admin closes it.

## Schedule

| Frequency | Window | Opens | Closes | Fiscal year label |
|---|---|---|---|---|
| Quarterly | Q1 | Jan 1 | Jan 30 | FY = calendar year (Jan 2027 opens FY27 Q1) |
| Quarterly | Q2 | Apr 1 | Apr 30 | FY = calendar year |
| Quarterly | Q3 | Jul 1 | Jul 30 | FY = calendar year |
| Quarterly | Q4 | Oct 1 | Oct 30 | FY = calendar year |
| Annually | Full year | Oct 1 | Oct 30 | FY = calendar year (Oct 2026 opens FY26) |

Values written to Dataverse: `ecm_reportingfrequency` = `Quarterly` or `Annually`, `ecm_fiscalyear` = `FY26`,
`ecm_quarter` = `Q1`..`Q4` (blank for Annually), `ecm_status` = `Open` or `Closed`.

## Flow definition

**Trigger:** Recurrence, every 1 day, 06:00, time zone `(UTC-05:00) Eastern Time (US & Canada)`.

**Connection:** Dataverse connector, run as a service account that has create/update on `ecm_reportingwindows`,
`ecm_measure` (read) and `ecm_measureactual`.

### 1. Compose date variables

| Name | Type | Expression |
|---|---|---|
| `today` | string | `convertFromUtc(utcNow(), 'Eastern Standard Time', 'yyyy-MM-dd')` |
| `day` | int | `int(formatDateTime(outputs('today'), 'dd'))` |
| `month` | int | `int(formatDateTime(outputs('today'), 'MM'))` |
| `fy` | string | `concat('FY', formatDateTime(outputs('today'), 'yy'))` |
| `quarter` | string | `if(equals(outputs('month'),1),'Q1', if(equals(outputs('month'),4),'Q2', if(equals(outputs('month'),7),'Q3', if(equals(outputs('month'),10),'Q4',''))))` |
| `inOpenDays` | bool | `lessOrEquals(outputs('day'), 30)` |

### 2. Compose `windowsToOpen` (array)

```
union(
  if(and(outputs('inOpenDays'), not(empty(outputs('quarter')))),
     createArray(json(concat('{"frequency":"Quarterly","fy":"', outputs('fy'), '","quarter":"', outputs('quarter'), '"}'))),
     createArray()),
  if(and(outputs('inOpenDays'), equals(outputs('month'), 10)),
     createArray(json(concat('{"frequency":"Annually","fy":"', outputs('fy'), '","quarter":""}'))),
     createArray())
)
```

### 3. Apply to each `windowsToOpen` → open the window

1. **List rows** `ecm_reportingwindowses`, filter:
   - Quarterly: `ecm_reportingfrequency eq 'Quarterly' and ecm_fiscalyear eq '@{items('w')?['fy']}' and ecm_quarter eq '@{items('w')?['quarter']}' and statecode eq 0`
   - Annually: `ecm_reportingfrequency eq 'Annually' and ecm_fiscalyear eq '@{items('w')?['fy']}' and statecode eq 0`
2. **Condition** `empty(outputs('List_window')?['body/value'])`
   - Yes → **Add a new row** `ecm_reportingwindowses`: `ecm_reportingfrequency`, `ecm_fiscalyear`, `ecm_quarter` (omit for Annually), `ecm_status` = `Open`.
   - No → if `first(...)?['ecm_status']` is not `Open` → **Update a row** `ecm_status` = `Open`.
3. **Condition** `equals(items('w')?['frequency'], 'Quarterly')` → Yes:
   1. **List rows** `ecm_measures`, filter
      `statecode eq 0 and (ecm_reportingfrequency eq 'Quarterly (Count)' or ecm_reportingfrequency eq 'Quarterly (Year-to-Date)')`
   2. **Apply to each** measure:
      1. **List rows** `ecm_measureactuals`, filter
         `_ecm_measure_value eq @{items('m')?['ecm_measureid']} and ecm_fiscalyear eq '@{items('w')?['fy']}' and ecm_quarter eq '@{items('w')?['quarter']}'`
      2. If empty → **Add a new row** `ecm_measureactuals`:
         - `ecm_measure@odata.bind` = `/ecm_measures(@{items('m')?['ecm_measureid']})`
         - `ecm_fiscalyear` = fy, `ecm_quarter` = quarter
         - `ecm_target` = `items('m')?['ecm_target']`
         - `ecm_status` = `true` (Yes = open for editing)
      3. Else if `ecm_status` is false → **Update a row** `ecm_status` = `true`.

   Annual measures are not pre-created. Bureau users create the row with **New**, which the page only shows while
   an Annual window is open and no row for that FY exists yet.

### 4. Close windows whose scheduled end has passed

1. **List rows** `ecm_reportingwindowses`, filter `ecm_status eq 'Open' and statecode eq 0`.
2. **Apply to each** window `x`:
   1. Compose `endMonth`:
      `if(equals(items('x')?['ecm_reportingfrequency'],'Annually'), 10, if(equals(items('x')?['ecm_quarter'],'Q1'),1, if(equals(items('x')?['ecm_quarter'],'Q2'),4, if(equals(items('x')?['ecm_quarter'],'Q3'),7,10))))`
   2. Compose `endDate`:
      `formatDateTime(concat('20', substring(items('x')?['ecm_fiscalyear'], 2, 2), '-', formatNumber(outputs('endMonth'), '00'), '-30'), 'yyyy-MM-dd')`
   3. Compose `daysSinceEnd`: `div(sub(ticks(outputs('today')), ticks(outputs('endDate'))), 864000000000)`
   4. **Condition** `and(greaterOrEquals(outputs('daysSinceEnd'), 1), lessOrEquals(outputs('daysSinceEnd'), 3))` → Yes:
      1. **Update a row** window `ecm_status` = `Closed`.
      2. **List rows** `ecm_measureactuals`, filter
         - Quarterly: `ecm_fiscalyear eq '@{items('x')?['ecm_fiscalyear']}' and ecm_quarter eq '@{items('x')?['ecm_quarter']}' and ecm_status eq true`
         - Annually: `ecm_fiscalyear eq '@{items('x')?['ecm_fiscalyear']}' and ecm_quarter eq null and ecm_status eq true`
      3. **Apply to each** → **Update a row** `ecm_status` = `false`.

   Windows opened by hand outside the schedule have `daysSinceEnd` far from 1..3 and are left alone.

### 5. Failure handling

- Set the flow's **Retry policy** on each Dataverse action to Exponential, 4 retries.
- Add a parallel branch configured to **run after: has failed, has timed out** that sends an email to the TOPS admin
  mailbox with `workflow()?['run']?['name']`.
- Because the close step accepts days 1 to 3 after the end date, a single missed run self-heals the next morning.

## Page behaviour that depends on this flow

| Page | Behaviour |
|---|---|
| `measure-detail-user.html` | Banner shows the open window or the schedule. **New** appears for Annual measures only while an Annual window is open and no row exists for that FY. FY and quarter are pre-selected and locked to the open window. Quarterly rows are editable only while `ecm_status` is true, which this flow controls. |
| `measure-detail-admin.html` | Same banner. **New** always available; FY and quarter pre-selected to the open window but not locked. |
| `reporting-windows.html` | Automated Schedule table with the next occurrence of each window. Toggles remain as the manual override. |

Power Pages caches Liquid FetchXML output for up to 15 minutes, so a window opened at 06:00 can appear on the
pages up to 15 minutes later.

## Test plan

1. In dev, set the recurrence to run now, with `today` temporarily hard-coded to `2027-01-01`. Confirm one
   `ecm_reportingwindows` row `Quarterly / FY27 / Q1 / Open` and one open Actual row per active quarterly measure.
2. Hard-code `2027-01-31`. Confirm the window is `Closed` and those Actual rows have `ecm_status` = false.
3. Hard-code `2026-10-01`. Confirm both `Quarterly / FY26 / Q4` and `Annually / FY26` open.
4. Open a window by hand for `FY25 Q2`, run the flow with any date. Confirm it stays open.
5. Remove the hard-coded date before enabling the flow.
