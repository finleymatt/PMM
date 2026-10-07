# Reporting windows (browser-side, no flow)

The measure detail pages compute the open reporting window from today's date. No scheduled flow is required.

## Calendar

| Frequency | Period | Open | FY label |
|---|---|---|---|
| Quarterly | Q1 | Jan 1 - 30 | calendar year (Jan 2027 = FY27 Q1) |
| Quarterly | Q2 | Apr 1 - 30 | calendar year |
| Quarterly | Q3 | Jul 1 - 30 | calendar year |
| Quarterly | Q4 | Oct 1 - 30 | calendar year |
| Annually | Full year | Oct 1 - 30 | calendar year (Oct 2026 = FY26) |

## Override

An `ecm_reportingwindows` record for a period wins over the calendar. Switch a window **On** on Manage Reporting
Windows to keep it open outside its dates, or **Off** to close it early. Periods with no record follow the calendar.

## What the pages do

| Page | Behaviour |
|---|---|
| `measure-detail-user.html` | Banner shows the open window or the calendar. A row is editable only while its `ecm_status` is open **and** its window is open. Annual: **New** shows only while the Annual window is open and no row exists for that FY; FY is pre-selected and locked. Quarterly: the first visit during an open window creates the row for that quarter with `ecm_status` = true and the measure target copied in. |
| `measure-detail-admin.html` | Same banner and auto-create. New, edit and delete stay unrestricted; FY and quarter are pre-selected but not locked. |
| `reporting-windows.html` | Automated Schedule table with the next occurrence of each window; toggles are the override. |

Auto-create does a live Web API `GET` for the quarter first, so two users opening the same measure do not create
two rows, and marks the browser session so a stale page cache cannot create a second one.

## Prerequisites in Power Pages

1. Site setting `Webapi/ecm_measureactual/enabled` = `true` (already in use for New and Edit).
2. Site setting `Webapi/ecm_measureactual/fields` must include `ecm_status` and `ecm_target` in addition to the
   fields already listed, or be `*`.
3. Table permission for the PMM User web role on `ecm_measureactual` must include Create (already required by the
   Annual New button) and Read.
4. `ecm_status` on `ecm_measureactual` is written as boolean `true`. If the column is a text or whole-number
   column, change the one line `ecm_status: true` in `ensureOpenWindowRow()` on both pages to match.

## Limits

- A quarterly row is created when someone first opens the measure during the window, not at 00:00 on the 1st.
  Rows that nobody opens are created by the first visitor, including an admin.
- Dates come from the visitor's browser clock.
- Power Pages caches Liquid output for up to 15 minutes, so a row created on first visit may take that long to show
  in the table. The banner says so when that happens.
