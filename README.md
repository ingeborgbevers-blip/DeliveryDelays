# Outstanding Orders, Changing Delivery Promises & Recorded Activity

![Power BI Dashboard](Delivery%20Promises%20and%Recorded%20Activity.png)

## Overview

This project investigates outstanding order quantities, revised delivery promises and recorded order activity in a fictional SME distribution business.

Using an entirely synthetic dataset and Power BI, the analysis follows a practical operational question: **which outstanding commitments need a closer look, even when the latest promised delivery date has not yet passed?**

The case study develops over two stages. The first identifies order lines with extended delivery promises and long gaps since recorded activity. The second follows selected lines into their event histories to examine the recorded explanations and identify further checks.

The workflow is:

Synthetic Excel / CSV tables → Power Query preparation → Power BI relationships and DAX measures → exception visuals → event-history investigation → proposed operational actions

The project is relevant to distribution, manufacturing and warehousing SMEs, particularly where customer service, planning and finance rely on ERP order reports.

**All businesses, transactions, events and references are fictional. Findings demonstrate an analytical approach; they are not results achieved for a client.**

---

## The Problem

An order can remain open against a delivery promise that has moved substantially since the order was placed. A report showing only the latest promise and current status may leave the reader without enough context to decide what needs attention.

The investigation asks:

* How much remains outstanding after deliveries and cancelled quantities?
* How far has the current delivery promise moved from the original commitment?
* How long has it been since the last recorded order activity?
* What does the change history say the order was waiting for?
* Can the next person reviewing the order identify the current position and next action?

An overdue date is a flag requiring context. Equally, a revised date in the future does not, by itself, explain the age of the original commitment.

> A reason code explains the last update. Does it tell the next person what to do?

---

## The Approach

### Dataset and scope

The dataset covers orders created between July and September 2026, with a fixed reporting snapshot of **30 September 2026**.

| Population | Count |
|---|---:|
| Customers | 24 |
| Products | 18 |
| Sales orders | 400 |
| Order lines | 803 |
| Delivery-line records | 784 |
| Recorded history events | 2,653 |

Quantities are measured in **cases**. Prices and values are in **GBP excluding VAT**. Dates have no time-of-day component.

### Stage 1: Outstanding quantities and reporting flags

The first stage compares remaining quantities, original and current promises, and the most recent recorded activity date.

The main scatter plot uses:

* one bubble per outstanding order line;
* horizontal position for promise extension in days;
* vertical position for days since the last recorded activity;
* bubble size for outstanding quantity;
* colour for current order-line status.

Supporting tables and charts examine the interval between the latest promise change and last recorded activity, and the interval from order creation to the latest promise change.

These intervals describe the sequence of events. A larger interval is not automatically evidence of worse service, and the latest promise change is not necessarily the first response to an issue.

### Stage 2: Following selected lines into their histories

Three open order lines were selected for further investigation. Their histories were reviewed for event dates, initiating roles, recorded reasons, event types, and old and new values.

The investigation distinguishes what the records demonstrate from what would need customer communication, procurement or other supporting information.

---

## The Solution

### Power Query and data preparation

The source is supplied as an Excel workbook and a CSV package. The core analytical tables are:

| Table | Grain and purpose |
|---|---|
| `Customers` | One row per customer |
| `Products` | One row per product |
| `OrderLines` | One row per sales-order line; quantities, promises, status and snapshot fields |
| `Deliveries` | One row per delivery line; multiple deliveries can relate to one order line |
| `LineHistory` | One row per recorded event; multiple events can relate to one order line |

The supplied dictionary describes the fields. Numeric quantities and prices must be imported as numeric types, and date columns as dates. Formatting a text quantity as a number does not convert its underlying type.

### Power BI model

Use active one-to-many relationships with single-direction filtering from the one side:

| One side | Many side | Key |
|---|---|---|
| `Customers` | `OrderLines` | `CustomerID` |
| `Products` | `OrderLines` | `ProductID` |
| `OrderLines` | `Deliveries` | `OrderLineID` |
| `OrderLines` | `LineHistory` | `OrderLineID` |

Keep deliveries and history as separate child tables. Expanding both into one flat table can multiply delivery quantities because an order line may have several deliveries and several events.

Use `OrderLines[OrderLineID]` for the row or bubble identifier in an order-line overview. This preserves open lines that have no delivery record.

### Core calculations

The following measures assume queries are named `OrderLines` and `Deliveries`:

```DAX
Delivered Qty =
COALESCE(SUM('Deliveries'[DeliveredQty]), 0)
```

```DAX
Outstanding Qty =
SUM('OrderLines'[OrderedQty])
    - SUM('OrderLines'[CancelledQty])
    - [Delivered Qty]
```

At order-line level:

```text
Promise extension (days) = CurrentPromiseDate - OriginalPromiseDate
Days since recorded activity = SnapshotDate - LastActivityDate
Activity after latest promise change = LastActivityDate - latest PromiseChanged event date
Time to latest promise change = latest PromiseChanged event date - OrderDate
```

Lines with no promise-change event should remain blank for the two event-based intervals, rather than being assigned zero.

For an overdue calculation, specify the promise baseline. A positive overdue duration means the promise date is before the snapshot; a future promise is not overdue. Always use the supplied snapshot rather than today's date when reproducing this analysis.

---

## Dashboard Highlights

### Outstanding orders and changing promises

The overview brings outstanding quantity, promise extension and recorded inactivity together. It supports selecting lines for investigation rather than automatically classifying them as service failures.

One illustrative line, `SO00052-1`, shows:

* original delivery promise: **7 July 2026**;
* current delivery promise: **5 October 2026**;
* promise extension: **90 days**;
* outstanding quantity: **80 cases**;
* days since last recorded activity: **86**.

At the 30 September snapshot, its current promise was still five days away. The original commitment and activity history nevertheless provide a reason to review the remaining balance.

### Selected history investigation

The second stage examines these three open lines:

| Order line | Promise extension | Last recorded activity | Days since activity | Reason at latest review |
|---|---:|---|---:|---|
| `SO00052-2` | 87 days | 13 July 2026 | 79 | Awaiting customer instruction |
| `SO00360-2` | 77 days | 20 July 2026 | 72 | Awaiting stock |
| `SO00388-1` | 80 days | 20 July 2026 | 72 | Awaiting customer instruction |

All three histories record a **Supply reschedule** promise change before the subsequent review. These are selected examples, not a representative sample of all orders.

---

## Key Findings

### 1. The latest promise needs its original context

A current promise date can be accurate while leaving the reader unaware of a substantial extension. Keeping both dates visible supports a more informed review of the outstanding commitment.

### 2. Outstanding quantity belongs at order-line level

Partial deliveries do not establish completion. Delivered quantities must be aggregated and cancelled quantities deducted before the remaining balance is assessed.

### 3. Recorded inactivity identifies questions

The selected lines had 72–79 days since their latest recorded activity. This establishes a gap in the available order-line history, not the absence of customer conversations or work elsewhere.

### 4. A recorded reason does not necessarily identify the next action

“Awaiting customer instruction” and “Awaiting stock” explain the recorded position at the latest review. Further information is needed to establish whether that position remains current, who owns the follow-up and when another review is due.

### 5. Automatic updates need separate interpretation

The history contains system status events as well as human reviews and delivery events. `LastActivityDate` therefore does not measure the date of last customer contact.

---

## Operational Recommendations

These are proposed actions to test. The case study contains no evidence of implementation or measured improvement.

### 1. Make supporting information traceable

Check where customer instructions, revised commitments and supply updates are recorded. Provide a reference from the order line so staff can find the latest supporting information. Assess the existing workflow before deciding whether system integration is needed.

### 2. Give each exception an owner and a review date

Record the responsible person, outstanding action and next review date. An old reason code should prompt a check of the current position rather than an assumption that the previous explanation still applies.

### 3. Agree follow-up and escalation rules

Use outstanding quantity, promise changes and review dates to support an exception view. Agree thresholds with the operational team and work through examples so staff understand what to check, when to contact another team and when to escalate.

---

## Data Limitations

* The dataset is entirely synthetic and does not establish the prevalence of these patterns in real businesses.
* Recorded reasons describe entries in the event history; they do not independently prove root causes.
* Customer-contact logs, complete procurement records and warehouse activity are outside this dataset.
* A blank customer-agreement reference is not proof that no agreement existed.
* `LastActivityDate` includes system events and must not be labelled “last customer contact”.
* The snapshot is fixed at 30 September 2026. Later outcomes are not modelled.
* Returns and over-deliveries are not modelled.
* The three selected lines do not support conclusions about every customer, product or order.

The analysis does not establish poor customer service, staff negligence or a verified service improvement. It identifies reporting flags, follows the available evidence and proposes practical checks.

---

## Demo

Suggested walkthrough:

1. Read the dataset scope and field definitions.
2. Review the outstanding-order table and scatter plot.
3. Select a line with a substantial promise extension and a long recorded activity gap.
4. Reconcile its ordered, delivered, cancelled and outstanding quantities.
5. Follow its promise changes and subsequent events in `LineHistory`.
6. Separate the demonstrated finding from questions requiring other records.
7. Review the proposed actions and the evidence needed to test them.

The Excel dataset is named `StraightLine_Delivery_Dataset.xlsx`. The alternative CSV package is named `StraightLine_Delivery_CSV_Pack.zip`. Accompanying case study PDF exports document the two investigation stages.

If publishing the CSV package for readers to investigate independently, review the optional `AUTHOR_KEY.md` before including it: it contains synthetic design benchmarks.

---

## How to Run

### Requirements

* Power BI Desktop
* The supplied Excel workbook or extracted CSV files

No SQL database is required for this version of the project.

### Steps

1. Download the dataset from the repository's published files.
2. In Power BI Desktop, choose **Get data → Excel workbook**, or import the five core CSV files using **Text/CSV**.
3. Select the core data tables and check their column types in Power Query.
4. Rename queries to match the table names used above, or adapt the DAX references to your query names.
5. Create the relationships described in the model section. Check that all use single-direction filtering.
6. Create the quantity measures and date-interval calculations.
7. Build the order-line overview using `OrderLines[OrderLineID]`, then filter the scatter plot to `[Outstanding Qty] > 0`.
8. Add promise extension to the horizontal axis, days since recorded activity to the vertical axis, outstanding quantity to bubble size and current status to the legend. Configure one point per order line.
9. Build a separate event-history table for selected lines.
10. Reproduce the selected examples using the 30 September 2026 snapshot and compare them with the case study.

---

## Suggested Business Use

This case study demonstrates a focused reporting investigation for businesses that need to connect outstanding commitments with the work recorded behind them.

It supports discussions about:

* retaining original commitments alongside revised promises;
* prioritising outstanding order lines for review;
* connecting planning and customer-service information;
* making follow-up ownership visible;
* and helping staff interpret exceptions consistently.

> A report becomes more useful when people can follow the figure back to the work that produced it—and understand what to do next.

---

## Tools Used

* Synthetic Excel and CSV data
* Power BI Desktop
* Power Query
* DAX
* PDF case study exports

Prepared by **StraightLine Data & Training**.

---

## Project Status

The two-stage illustrative analysis is complete. Operational recommendations remain proposals.

Potential future work includes adding customer-contact records, customer agreement evidence, purchase-order and supply updates, action ownership and review dates. A subsequent evaluation would be needed to establish whether any implemented changes reduce unresolved exceptions or improve customer communication.
