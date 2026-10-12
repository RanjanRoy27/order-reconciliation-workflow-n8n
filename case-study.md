# Meridian Supplies Practical Exercise — Work Completed to Date

## Where I got to

This is my progress on the Meridian Supplies exercise within the assessment time-box.

I focused first on getting the ingestion and data-cleaning part working properly before moving deeper into reconciliation and calculations. The data had quite a few edge cases, so I spent a good amount of time understanding what was actually happening to each record inside Make rather than rushing through the remaining parts.

So far, I completed the foundation for Source A and started Source B.

### What I worked on

- Source A CSV ingestion from Google Drive into Make.com
- Row-by-row processing using an Iterator
- Customer-name cleaning and normalisation
- Date normalisation
- Quantity, price and cost parsing
- SKU extraction from product descriptions
- Customer matching
- Validation and exception handling
- Idempotency using order reference + line number
- Airtable staging tables
- Source B JSON ingestion
- Source B order/item iteration
- Business-key generation and duplicate checking
- Reviewing the actual Airtable output and tracing records through the workflow
- Planning the reconciliation, calculation and Northgate parts

I didn't get to the final `results.csv`, full three-source reconciliation, shipment-event processing, FX calculation, credit checks, commission calculation or the Northgate build within the available time.

---

## How I used AI

I used AI quite a lot during this assessment, mainly as a development and debugging copilot.

I used it for things like:

- understanding the case-study requirements;
- going through the supplied data and finding possible issues;
- working out the Make.com module structure;
- writing and debugging Make formulas;
- figuring out customer-name normalisation;
- working through date parsing;
- thinking through regex/SKU extraction;
- designing the Airtable staging structure;
- working through idempotency and exception handling;
- checking why a particular Make module was producing an unexpected result;
- thinking through the reconciliation and calculation logic;
- and documenting the approach and remaining work.

The important part for me was that AI could suggest a direction, but I still had to put it into Make, run it against the actual data, look at the bundles, find where things went wrong, and adjust the workflow.

That happened quite a few times during this exercise.

So I used AI more like a second pair of eyes while building and debugging, rather than treating its output as something that automatically worked.

The brief also explicitly allows AI tools, with the expectation that candidates can explain the decisions they made. That's how I approached it.

---

# What I actually built

## 1. Source A ingestion

I started with the warehouse CSV.

The workflow retrieves the file from Google Drive, parses the CSV, and sends each row through an Iterator in Make.

I then created separate variables for the fields I needed to work with instead of carrying the raw row through the whole scenario.

The main fields I worked with were:

- `order_ref`
- `line_no`
- `order_date`
- `customer_name`
- `product_description`
- `quantity`
- `unit_price`
- `cost_price`
- `currency`

From there, I started building a cleaned version of each row.

---

## 2. Data cleaning

This ended up taking more time than I initially expected.

The source data contains several cases where the same type of information is represented differently, so I wanted to clean and standardise those values before doing any reconciliation.

### Customer names

I normalised customer names by trimming whitespace, converting to lowercase and removing selected punctuation.

The basic idea was:

`lower(trim(remove selected punctuation))`

I kept this separate from the original customer name so the original data could still be retained while the cleaned version could be used for matching.

I also looked at customer reference/customer email as another way of confirming a match.

### The O'Brien problem

One of the more annoying cases was `O'Brien`.

The name itself was completely valid, but the apostrophe became a problem when the cleaned value was inserted into an Airtable search/filter expression.

So I ended up separating:

- the normalised customer name used for matching;
- the search-safe version used when sending the value into the Airtable query.

This was a good example of something that looked like a simple text-cleaning problem but became a query/escaping problem later in the workflow.

### Dates

The Source A data uses more than one date format.

I worked on a parser that could handle:

- `YYYY-MM-DD`
- `DD/MM/YYYY`

and convert them into a consistent `YYYY-MM-DD` format.

That becomes especially important later because the order date is also used for the historical currency lookup.

### Quantity

Some quantity values contained text such as `units`.

I removed the text and converted the remaining value into a number.

### Prices and costs

I cleaned comma-separated numeric values before converting them to numbers so that they could be used safely in later calculations.

### SKU extraction

The SKU wasn't always provided as a separate clean field. In some cases it was embedded inside the product description.

I therefore added a separate step to extract and normalise the SKU.

I also ran into an encoding/display issue where a special character in a product description appeared as `�` in Make even though the source file contained the correct character.

That was another thing I had to investigate rather than immediately assuming the source data was broken.

---

# 3. Validation and exceptions

I wanted the workflow to be able to deal with bad rows without the whole run stopping.

The validation logic started covering things such as:

- invalid currency;
- missing cost price;
- negative/invalid quantity;
- date problems;
- customer matching issues;
- duplicate order lines;
- and values that could not safely be converted.

For example, during the current runs I found:

- `MRD-00414` — invalid currency;
- `MRD-00421` — missing cost price;
- `MRD-00411` — negative quantity that still needed to be reconciled with the rejection logic.

The intended flow is basically:

**find the problem → record it → don't process the bad value → keep the rest of the run going.**

That matches the requirement that one bad row shouldn't bring down the whole scenario.

---

# 4. Idempotency

I also worked on making sure the same order line doesn't get written twice if the scenario is run again.

For Source A, I used:

`cleaned order_ref + line_no`

as the business key.

The line number matters because Source A is line-item data. The order reference by itself isn't enough to uniquely identify every row.

The workflow checks for the key before writing the record.

The goal is to safely handle:

- rerunning the same file;
- running Day 2 after Day 1.

---

# 5. Source B ingestion

Once I had the Source A foundation working, I moved to the e-commerce JSON.

The Source B workflow:

1. gets the JSON from Google Drive;
2. parses it;
3. iterates through the orders;
4. iterates through the items;
5. normalises the order reference and SKU;
6. creates a business key;
7. checks Airtable for an existing key;
8. writes a new staging record when the key doesn't already exist.

One useful thing I found was that Source B contains orders such as `MRD-00440` that aren't present in the current Source A data.

That shouldn't just disappear from the workflow. It needs to show up later as a Source B-only case.

---

# Problems I ran into

This was probably the most useful part of the exercise for me because quite a few things didn't work on the first attempt.

## Make.com credits

The biggest practical issue was the Make operation/credit limit.

I had 33 Source A rows to test, and every full run consumed operations. Because I was changing and testing the scenario repeatedly, the credits disappeared much faster than expected.

Eventually I couldn't keep doing full end-to-end runs every time I changed something.

So I switched to checking individual modules, looking at the bundle output, fixing the issue, and only doing a full run when I thought the change was ready.

That definitely slowed things down, especially because this particular exercise has a lot of edge cases where repeated testing is useful.

## Airtable update error

I also hit this Airtable error:

> Cannot update more than one record for fields to merge on

The problem was that my Airtable operation wasn't resolving to one unique record.

That forced me to look more carefully at the uniqueness of the matching field instead of assuming that the value would always identify one record.

It was a good reminder that a workflow can look correct on the surface but still have a logical problem in how it finds records.

## O'Brien / search escaping

The `O'Brien` case was another debugging problem.

The customer name was valid. The issue happened when that value was used inside an Airtable search expression.

This led me to separate normalised data from search-safe data.

## Special-character / encoding issue

I also had a product description where a special character showed up as `�` in Make.

I compared the source data with the Make output to figure out whether the issue was in the original file or somewhere in the transformation.

I didn't want to just replace the character without understanding where it changed.

## Make formula issues

I spent quite a bit of time getting the Make formulas right.

Some formulas that look straightforward in normal programming syntax don't translate directly into Make's formula/mapping syntax.

Instead of trying to make one huge cleaning formula, I started breaking the process into smaller variables:

- cleaned customer name;
- search-safe customer name;
- cleaned SKU;
- cleaned quantity;
- cleaned price;
- ISO order date.

That made it much easier to see exactly where a value was changing.

---

# What I would have done next

Once the ingestion layer was stable, I would have moved into the final order-level pipeline.

## Step 1 — Finish Source A

First I would:

- finish the remaining validation cases;
- complete the exception records;
- verify all 33 input rows;
- check the accepted/rejected counts;
- run the same input again to verify idempotency.

## Step 2 — Reconcile Source A + Source B

Next:

- normalise the order references;
- match orders and line items;
- compare conflicting values;
- keep both values when there is a conflict;
- record the conflict in the exceptions output;
- identify Source B-only orders;
- populate `sources_seen`;
- populate `conflict_flag`.

## Step 3 — Add Source C

Source C is slightly different because shipment events can arrive out of order and some events are duplicated.

I would deduplicate the events and use the event timestamp to determine the current shipment status rather than simply using whichever event arrived last.

That prevents an old event arriving late from overwriting the current status.

## Step 4 — Add historical FX

I would then use Make's HTTP module to call Frankfurter and retrieve the historical exchange rate for the order date.

For weekends or dates without a published rate, I would use the most recent previous published rate and record that fallback in the exceptions output.

I would also add an error route for an unavailable or unexpected API response.

## Step 5 — Build the final order calculations

After reconciliation, I would aggregate the line items into one row per order and calculate:

- original subtotal;
- FX rate;
- subtotal in EUR;
- discount;
- net EUR;
- VAT;
- total EUR;
- cost EUR;
- margin percentage;
- credit check;
- final status.

## Step 6 — Credit checks

I would compare the customer's existing outstanding balance plus the new order against their credit limit.

If the order pushes them over the limit, it would go onto a hold path and create an exception.

## Step 7 — Commission

Finally, I would calculate monthly gross-margin contribution per sales rep, apply the progressive commission rates, handle split orders, and apply cancelled-order clawbacks.

## Step 8 — Northgate

After the Meridian workflow was stable, I would adapt the same structure for Northgate.

I would keep the common parts reusable and change the client-specific mapping, date parsing and discount rules rather than rebuilding the whole scenario from scratch.

---

# Completed vs. remaining

| Area | Status |
|---|---|
| Source A ingestion | Completed foundation |
| CSV parsing | Completed |
| Iterator-based processing | Completed |
| Customer cleaning | Completed/iterated |
| Date normalisation | Implemented and tested |
| Quantity/price cleaning | Implemented |
| SKU extraction | Implemented |
| Basic validation | Implemented |
| Exception routing | Started |
| Source A idempotency | Implemented foundation |
| Source B ingestion | Implemented foundation |
| Source B idempotency | Implemented foundation |
| Final Source A/B reconciliation | Not completed |
| Source C shipment events | Not completed |
| Historical FX lookup | Not completed |
| Final order calculations | Not completed |
| Credit checks | Not completed |
| Commission | Not completed |
| Final `results.csv` | Not completed |
| Final `exceptions.csv` | Partially completed |
| Final `commission.csv` | Not completed |
| Northgate replication | Not completed |
| Part 4 written answers | Not completed |
| Part 5 SQL | Not completed |
| Part 6 written work | Not completed |

---

# Why I stopped here

The brief itself says the exercise is deliberately bigger than the available time and that prioritisation is part of the assessment.

So I decided to spend the available time getting the data foundation into a reasonable state rather than rushing through every part.

My thinking was simple:

**If the data going into the calculations isn't reliable, there's not much value in having a finished-looking calculation layer.**

I would rather have a working ingestion/cleaning foundation with known issues and a clear next step than rush the last few parts and not have enough time to test whether the numbers are actually right.

The remaining work is mostly building on the structure that is already there, rather than starting the whole solution again.

---

# My take on the build

I think the strongest part of what I built is the ingestion and cleaning side.

I spent a lot of time looking at the actual data, finding the weird cases, and figuring out why a value changed between modules.

The weakest part is where the staging data needs to become the final order-level result.

I didn't get enough time to complete and repeatedly test that part, so I wouldn't claim that the final numerical output is production-ready yet.

If I had more time, my next priority would be:

**stabilise the staging → canonical order layer → finish the three-source reconciliation → add calculations → run the complete pipeline → independently check the final numbers.**

---

# Final note

I didn't finish the full assessment within the time-box.

But I did get through the part where I was learning the data, building the Make workflow, cleaning the records, handling edge cases, and debugging real failures.

A lot of the remaining work is already mapped out from the current structure.

So if I were continuing, I wouldn't restart the build. I'd continue from the current staging layer and work downstream from there.
