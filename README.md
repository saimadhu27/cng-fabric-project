# CNG Distribution Analytics Platform

This project simulates the data pipeline for a B2B wholesale distribution business, modeled loosely on a company that buys and resells paper, packaging, and related products through a fairly typical multi-system ERP setup.

It's built on Microsoft Fabric using a medallion architecture: raw data comes in, gets cleaned up, and lands as a star schema ready for Power BI. Along the way there are data quality checks that will actually stop the pipeline if something looks wrong, rather than just letting bad data flow through.

## Table of Contents

1. [What the project does](#1-what-the-project-does)
2. [Architecture](#2-architecture)
3. [Data model](#3-data-model)
4. [What happens at 10x the data](#4-what-happens-at-10x-the-data)

## 1. What the project does

The source data is five CSV extracts meant to look like they came from a blended Oracle + SQL Server environment: Purchase Orders, Sales Orders, Products, Customers, and Inventory. About 6,700 rows total.

That raw data moves through three stages:

- **Bronze** - just captures what came in, unchanged
- **Silver** - cleans it up: fixes types, parses dates, standardizes categories, drops duplicates
- **Gold** - reshapes it into a star schema for Power BI

Each stage checks the data before passing it to the next one, and if something fails the check, the pipeline stops there instead of pushing bad data further down the line.

Here's roughly how many rows survive each stage:

| Table | Raw rows | Rows in Gold |
|---|---|---|
| Sales orders | 5,150 | 4,864 |
| Purchase orders | 1,248 | 1,144 |
| Inventory | 215 | 187 |
| Products | 42 | 41 (includes an "Unknown" row) |
| Customers | 72 | 71 (includes an "Unknown" row) |

Stack: Python and pandas for the transformations, Fabric Data Factory to orchestrate everything, Great Expectations for data quality, Power BI on the front end.

## 2. Architecture

### The three layers

**Bronze** is the raw landing zone. Every row gets stamped with when it arrived, which file it came from, and which pipeline run brought it in. Nothing gets changed here - it's just a clean record of what actually showed up.

**Silver** is where the real cleanup happens. Dates get parsed properly, text fields get normalized, required fields that are missing get dropped, and duplicate rows get removed.

**Gold** takes the cleaned Silver data and reshapes it into a star schema - dimension tables and fact tables, joined properly, with every fact row pointing to something real in a dimension table.

Keeping these three stages separate turned out to matter more than expected. At one point there was a bug in how Silver title-cased certain text fields, and fixing it only meant re-running Silver - Bronze didn't need to be touched. If everything had been one big script, that fix would've meant starting over from scratch.

### Why pandas instead of Spark

This runs on pandas rather than Spark, and writes plain CSV files instead of Delta tables. Worth being upfront about what that costs:

pandas runs on a single machine and holds everything in memory. It works fine for a dataset this size, but it doesn't get Spark's ability to spread work across a cluster - so it won't scale the same way if the data gets a lot bigger.

Writing CSVs instead of Delta tables means no built-in schema enforcement, no transaction safety, and every pipeline run rewrites the whole file instead of just adding what's new. It also means Power BI's Direct Lake mode isn't available without an extra step to load the data into a proper table first.


### How it's orchestrated

Fabric Data Factory runs the pipeline in order: Bronze, then a Bronze quality check, then Silver, then a Silver quality check, then Gold. Each step only runs if the one before it succeeded. If a quality check fails, the notebook raises an error, Fabric marks that step as failed, and nothing downstream runs. 

### Data quality checks

Between each layer, an automated check (using Great Expectations) looks at things like: did we get roughly the number of rows we expected, are all the columns present, are the required fields actually filled in. If something looks off, the pipeline stops instead of quietly passing the problem along.

## 3. Data model

The Gold layer is a standard star schema - three dimension tables and three fact tables, joined on the cleaned-up business keys (product ID, customer ID, date).

| Table | What one row represents |
|---|---|
| dim_product | One product |
| dim_customer | One customer |
| dim_date | One calendar day (covers 5 years) |
| fact_sales | One line on a sales order |
| fact_purchases | One line on a purchase order |
| fact_inventory | One inventory snapshot |

If a fact row doesn't have a matching product or customer, it doesn't just get dropped. It gets pointed at an "Unknown" row in that dimension instead. That way it still shows up in Power BI - as "Unknown Product," for example - rather than silently disappearing from a total with no indication anything's missing.


## 4. What happens at 10x the data

If this pipeline suddenly had to handle 10x the volume - around 50,000 sales rows, 12,000 purchase orders - it would still work. That's genuinely not a lot of data for pandas to handle on a single machine; it'd just take a bit longer to run.

The more interesting question is what starts to hurt even before it breaks outright:

Rewriting the entire CSV on every run starts to feel wasteful once most of the data isn't actually new. That's the point where switching to something that only adds or updates changed rows would clearly start paying off.

pandas running on one core also starts to leave performance on the table - Spark would spread that work across a cluster, but pandas just doesn't.

And the CSV-instead-of-Delta decision gets more expensive to live with as the files get bigger, since loading a larger CSV into a proper table before Power BI can use Direct Lake mode takes longer too.

None of that forces a rebuild at 10x, though. The point where this design would genuinely need to change is probably another order of magnitude out - where a single table stops comfortably fitting in memory, where the business actually needs incremental updates instead of full reloads, or where tracking history (like "what was this customer's credit limit last year") becomes a real requirement, since the current key structure doesn't support that at all.
