# Take-home Assessment

## Time

One hour. Stop when you hit it. If you run out, document where you stopped and why.

## Background

Bookkeepers categorize bank transactions into accounting categories. Most decisions repeat — the same vendor shows up week after week. A naive system calls an LLM on every transaction. That works but costs money and is slow. A smarter system caches what it has already learned and only escalates when it has to.

You are building a small version of that system.

## Files

- `transactions.csv` — 300 historical transactions with categories assigned. Columns: `vendor_raw`, `amount`, `date`, `memo`, `category`. This is your seed data.
- `holdout.csv` — 50 transactions without categories. Columns: `vendor_raw`, `amount`, `date`, `memo`. This is your test set.
- `categories.txt` — the 12 valid categories. Predictions must come from this list.

## Task

Build a 3-tier classifier in Python.

**Tier 1 — Exact-match cache.** Normalize the vendor string (lowercase, strip transaction codes like `#1234`, `*XYZ`, `STORE 0042`, etc.). Look up in a write-once cache seeded from `transactions.csv`. Hit → return cached category.

**Tier 2 — Embedding similarity.** Embed the normalized vendor (and memo, if you want) using any model — `sentence-transformers`, OpenAI embeddings, anything. Find nearest neighbor in the cache above a similarity threshold you choose. Hit → return that category.

**Tier 3 — LLM fallback.** Call any LLM (Groq is free; OpenAI, Anthropic, anything works). Pass the transaction and the 12 categories, take its answer. Then **add the `(normalized_vendor → category)` pair to the cache** before returning.

If you do not want to set up an LLM API, stub Tier 3 with a hardcoded dictionary or a constant default. Call out the stub in your README.

## Run on the holdout set and produce

1. `predictions.csv` — same as `holdout.csv` plus a `predicted_category` column and a `tier` column (1, 2, or 3) showing which tier resolved it.
2. `metrics.json` — accuracy, percentage of decisions resolved by each tier, and an estimated total cost using these unit costs:
   - Tier 1: $0.00001 per call
   - Tier 2: $0.0001 per call
   - Tier 3: $0.005 per call
3. `README.md` — at most one page. Cover: how you normalized vendors, how you picked the Tier 2 similarity threshold, three failure modes you actually observed in your output.

## Written question

Separate from the build, in 200 words or less:

> What breaks if the cache is mutable instead of write-once? Be specific.

Save as `written.md` in the repo.

## What we are not specifying

The vendor normalization rules and the Tier 2 threshold are deliberately not specified. Pick, justify in your README, move on. Do not email asking for clarification.

## Submission

Public GitHub repo or zipped folder containing:

- your code
- `predictions.csv`
- `metrics.json`
- `README.md`
- `written.md`

That is the entire deliverable.


--------------------------------------------------------------------------------------------------------

# transaction classifier

3-tier system that categorizes bank transactions. cache first, embeddings second, LLM fallback third.

## how to run

install dependencies:

    pip install -r requirements.txt

run the classifier:

    python classify.py

generates predictions.csv and metrics.json in the same directory.

## vendor normalization

i strip the following from vendor strings:

- #CODE patterns: store/transaction IDs like #4521, #SDF234
- *CODE patterns where CODE contains digits: order IDs like *RT4567, *KM2349. i keep *WORD when it is all letters (like *business, *payment) since those carry actual meaning
- single-letter + digits codes: workspace/session IDs like Slack's T0123ABC
- standalone 3+ digit numbers: store numbers, flight numbers, phone suffixes
- corporate suffixes: inc, llc, llp, corp, corporation

then collapse whitespace.

the goal is making the same vendor hit the same cache key every time. anything my rules miss gets caught by tier 2.

## tier 2 threshold

i went with 0.75. reasoning:

- too high (0.9+) and you barely catch anything beyond exact match
- too low (0.5) and you start pulling false matches. short strings like "uber" match "united" because they are both short travel words
- 0.75 catches clear variants ("doordash.com" to "doordash *order") without crossing category boundaries

could tune this properly with labeled validation data. 0.75 was a judgment call.

## tier 3 stub

stubbed with keyword matching instead of an actual LLM. in production i would use gpt-4o-mini or claude haiku, cheap, fast, good enough for categorization. the stub still adds results to the cache so future lookups for the same vendor skip the expensive call.

## failure modes i observed

1. vendor variants that normalize differently. "MSFT*M365 BUS" normalizes to "msft bus" but "MSFT * OFFICE 365" normalizes to "msft * office". same company, different keys. tier 2 has to rescue this and it is not guaranteed to clear the threshold.

2. ambiguous vendors. "AMAZON.COM" could be Office Supplies or Equipment & Hardware depending on what was ordered. my cache takes the first category seen in training data. write-once means we are locked to that even if later transactions are different.

3. completely novel vendors. B&H Photo has zero presence in training data. if embeddings do not find a close match, it falls to the keyword stub which is fragile. this is exactly where a real LLM earns its cost.

## time spent

around whole 1 hr. i wrote readme and written.md after the one hour fininshed. most of the time went to normalization regex and making sure the holdout edge cases were handled. did not get to proper threshold tuning with cross-validation.
