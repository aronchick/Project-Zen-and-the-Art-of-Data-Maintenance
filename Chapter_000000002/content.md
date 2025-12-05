# Chapter 2: Data Types and the Structure Spectrum

## Or: Why Your Model Thinks a ZIP Code is a Really Big Number

**A model in production started predicting that everyone in Beverly Hills (90210) was exactly 902.1 times more likely to default on loans than people in Anchorage (99501). Turns out, someone forgot to tell the model that ZIP codes aren't actually quantities you can multiply.**

Chapter 1 was philosophy and principles. This chapter is about what data actually *is* - the nuts and bolts that determine whether your pipeline processes information or generates expensive nonsense.

Data types are like ingredients. You can have the best recipe in the world, but mix up salt and sugar and you're fucked. Unlike cooking, you can't taste-test your model and start over. Well, you can, but it costs $50,000 in compute and your manager starts asking extremely uncomfortable questions about your decision-making process.

## 2.1 The Great Data Type Disaster

A major retailer built a demand forecasting model. It was beautiful. ResNet backbone, attention layers, the works. It could predict next month's toothpaste sales in Kissimmee, FL at 10 am on Tuesday, down to the tube.

One tiny problem.

Someone, somewhere, decided that product IDs should be treated as continuous variables. The model learned that product 10000 (toilet paper) was exactly twice as much product as product 5000 (diamonds).

The model started making predictions like "If we run out of product 5000, we should stock twice as much and sell product 10000 instead!"

Mathematically sound. And insane.

Because the model had been performing so well on validation data, no one thought to triple-check. One week later, they discovered they'd auto-ordered 50,000 units of toilet paper for their jewelry department.

What's the failure here? In many ways, this is a "happy case" - at least the pipeline didn't crash, the system didn't error out, and nobody woke up at 3 AM to figure out why the website was down.

On the other hand, this is the worst of all possible worlds. The error went through ALL the systems with no warnings. You're going to spend hours (days? months?>o&7?FSK!) debugging it because you're not getting ANY signal about what to do next.

## 2.2 Structured vs Unstructured: The Real Trade-offs

People love to talk about structured vs unstructured data like it's black and white. "Databases are structured! Images are unstructured!" Sure, and my desk is organized because I can see the surface in one corner.

The reality is a spectrum:

```
Perfectly Structured                                                  Complete Chaos
    |                                                                      |
    SQL → CSV → JSON → XML → HTML → PDF → Images → Video → Your Nephew's Crayon Drawings
         ↑           ↑                         ↑
    "structured"  "semi-structured"      "unstructured"
                  (the DMZ of data)
```

> **Visual Note**: *[Diagram opportunity: The Data Structure Spectrum with real examples placed along it]*

The key insight isn't where your data sits on this spectrum - it's understanding that **structure is a promise that can be broken**.

That CSV file? It promises columns will align. That promise gets broken every time someone opens it in Excel and saves it with "helpful" formatting. That JSON API? It promises consistent schemas. That promise gets broken when the upstream team "just adds a field real quick." That SQL table? It promises type safety. That promise gets broken when someone creates a VARCHAR(MAX) column called "misc_data" because they didn't want to think about schema design.

## 2.3 The Four Horsemen of "Structured" Data Failure

Data doesn't just fail; it actively conspires against you. The obviously unstructured mess? That's honest. Respectable, even. It walks in looking like a dumpster fire and delivers exactly the dumpster fire you expected. 

No, it's the *clean* data. The data that shows up in a pressed suit, shakes your hand, passes every validation check, and then three months later you discover it's been counting 'California' and 'CA' as two different states THE ENTIRE TIME.

### Horseman 1: The Structural Lie (Syntactic Failure)

This is the most basic betrayal. The data claims to be CSV, JSON, or XML, but it fundamentally violates the syntax of that format. Your parser doesn't even get to the content; it just dies.

**The tell:** Your script errors out immediately with a `ParserError`, `JSONDecodeError`, or similar before you can even inspect the data.

**The example:** You're given a "simple" CSV of product reviews.

```
product_id,user_id,rating,review_text
101,45,5,"This product is "great," I love it!"
102,48,4,"Good value for the price.
Note: User bought on sale."
103,51,1,Worst purchase ever.
```

This isn't a CSV; it's a trap. The first data row has an unescaped quote that breaks the string. The second row contains a newline character in the middle of a field, making the parser think it's a new record. The third row doesn't have quotes at all. It's chaos masquerading as columns.

### Horseman 2: The Type Trap (Semantic Failure)

The file parses! The columns are neat, the rows are consistent. You've survived the syntax war. Congratulations. Now you get to the real battle: semantic integrity.

In this failure mode, the data is structurally sound, but the data types are profoundly misleading. Your model will happily ingest these columns and learn complete nonsense.

**The tell:** The code runs without errors, but your model's predictions are bizarre. It starts making mathematical connections between things that have no quantitative relationship.

**The example:** A clean-looking dataset of customer information.

| user_id | zip_code | customer_tier | is_premium |
| :--- | :--- | :--- | :--- |
| 1001 | 90210 | 1 | "true" |
| 1002 | 10001 | 3 | "false" |
| 1003 | 60601 | 2 | "true" |

A lazy import script reads this and infers:
- `user_id`: `int`
- `zip_code`: `int`
- `customer_tier`: `int`
- `is_premium`: `string`

The model now learns that a user from Beverly Hills (90210) is quantitatively 9x "more" than a user from NYC (10001), and that `customer_tier` 3 is three times "worse" than `customer_tier` 1. It also has no idea what to do with the strings "true" and "false".

### Horseman 3: The Semantic Sinkhole (Value Failure)

You've done it. The structure is perfect. The types have been meticulously cast to `category`, `datetime`, and `bool`. You are a data wizard.

And yet, your data pipeline still delivers garbage. Why? Because while the types are correct, the *values themselves* are logically impossible.

**The tell:** Your applications can use your data, but elements make no sense, flag impossible conditions, and your groupings are all off. You find yourself asking, "How can a user have negative time on site?"

**The example:** A validated table of user activity.

| user_id | birth_date | signup_date | sessions_last_month | age |
| :--- | :--- | :--- | :--- | :--- |
| 201 | 1990-05-15 | 1988-01-20 | 15 | 35 |
| 202 | 1985-11-02 | 2024-03-10 | -5 | 39 |
| 203 | 1900-01-01 | 2023-09-01 | 12 | 999 |

Everything is correctly typed (`datetime`, `int`), but the data is a logical nightmare:
- **User 201** signed up two years *before* they were born.
- **User 202** had a negative number of sessions, a physical impossibility.
- **User 203** has an age of 999, a clear sentinel value that was never cleaned.

Your types are right, but your data is still telling lies.

### Horseman 4: The Schema Mirage (Over-structuring Failure)

This is the expert-level mistake. Worried about the first three horsemen, you spend weeks getting your schema exactly right. You take semi-structured or free-text data and force it into a rigid schema.

Your strict parser ends up being so brittle that it either breaks on valid inputs or, worse, throws away most of your data.

**The tell:** Your data pass-through percentage is small (anything less than 90% is a real issue), or the "structured" data you produce seems to be missing crucial information that you can see plainly in the raw source.

**The example:** You're extracting structured data from customer support chat logs.

```
"My order #G451-B was late. I'd rate the experience a 2/10. Please call me at 555-0123."
```

You write a regex to capture this:
```
Order: #([A-Z0-9\-]+), Rating: (\d/10), Phone: (\d{3}-\d{4})
```

Your parser works perfectly on that one message. Then it encounters:

- `"Where is my package G451B? This is a 1-star experience."` (Fails: different order format, no numeric rating)
- `"Rating this a 2 out of 10. My order was G451-B."` (Fails: different order of information)
- `"Call me back about G451-B. It was awful (2/10)."` (Fails: context is different)

By forcing a rigid schema onto conversational text, you've created a system that can only understand one exact phrasing. You've mistaken variability for chaos and, in trying to create order, have destroyed valuable information.

## 2.4 Making Peace with Unstructured Data

The key to dealing with unstructured data is a microcosm of your overall data strategy: **Start with what you're going to DO with what you're collecting.**

This principle challenges the common instinct to immediately structure and normalize all incoming data. Instead, it advocates for a pragmatic approach: only invest effort in structuring data when you have a clear, immediate use for it.

### The Hospital Records Lesson

A hospital system decided to "modernize" by converting 20 years of medical records into structured data. Budget: $2M. Timeline: 6 months.

**Month 1**: "We'll use OCR!" (Narrator: They would not use OCR.)

**Month 2**: Someone discovers doctors don't write, they scribble potential lawsuits.

**Month 3**: Discovered multiple languages.

**Month 4**: Discovered coffee stains count as "medical imaging."

**Month 5**: Discovered records from the 80s were on carbon paper.

**Month 6**: Hired 50 medical students to manually transcribe.

**Final cost**: $8M
**Final timeline**: 18 months
**Accuracy**: "Good enough for billing"

The lesson? Most of that imagery should have just remained as scans. The only things this particular scenario needed were procedure codes, patient IDs, and dates. Sometimes unstructured data should stay unstructured, and you should build your system around that reality.

### The ROI of Structure

Before you structure anything, run this calculation:

| Data Type            | Storage Cost | Processing Time | Accuracy Potential | Variability |
| -------------------- | ------------ | --------------- | ------------------ | ----------- |
| Clean SQL            | $            | Minutes         | 95%                | Very low    |
| Dirty CSV            | $            | Hours           | 85%                | Low         |
| JSON                 | $$           | Hours           | 85%                | Low         |
| PDF Tables           | $$$          | Days            | 70%                | Medium      |
| Scanned PDFs         | $$$$         | Weeks           | 50%                | High        |
| Doctor's Handwriting | $$$$$        | Months          | Prayer             | ∞           |

If the equation doesn't turn out to be ROI positive, it's **perfectly acceptable** - even preferable - to leave data in its raw, unstructured form until you actually need it. Add some metadata about the ingestion and a pointer to the raw form, and you're already on a successful path.

### An E-Commerce Example: Selective Structure

Consider an e-commerce platform collecting user interaction data. The business has identified that purchase amount and product category are critical for immediate reporting, while other event details might be useful later but have no current defined use case.

```yaml
# User Event Schema
event:
  # Structured fields - explicitly needed for reporting
  structured:
    event_type: "purchase"           # Enum: purchase, view, cart_add, cart_remove
    timestamp: "2024-03-15T10:30:00Z" # ISO 8601 format, always UTC
    user_id: "usr_123456"             # Validated format: usr_[0-9]{6}
    
    # Purchase-specific structured data
    purchase:
      amount_cents: 4999              # Integer, validated range: 0-10000000
      currency: "USD"                 # Enum: USD, EUR, GBP, JPY
      product_category: "electronics"  # Enum from defined taxonomy
      payment_method: "credit_card"    # Enum: credit_card, paypal, apple_pay
    
  # Unstructured fields - kept raw for future use
  metadata:
    # Everything else goes here as-is, no validation or structure imposed
    device_info:
      raw_user_agent: "Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)..."
      screen_resolution: "1170x2532"
      connection_type: "4g"
    
    session_context:
      referrer: "https://google.com/search?q=wireless+headphones"
      campaign_params: "utm_source=google&utm_medium=cpc&utm_campaign=spring_sale"
      ab_test_variants: ["checkout_v2", "recommendation_algo_b", "nav_redesign"]
    
    product_details:
      sku: "WH-1000XM5-BLK"
      brand: "Sony"
      model_year: "2023"
      warehouse_location: "NJ-03"
      supplier_batch: "2023Q4-1847"
```

**Structured fields** are those we need immediately for financial reporting, conversion funnel analysis, payment processing. These have strict validation rules, defined enumerations, and clear data types.

**Unstructured metadata** includes everything else that might be valuable but lacks a current use case - device information, campaign parameters, product details we're not yet analyzing.

When a new use case emerges - say, analyzing conversion rates by screen resolution - we can promote `screen_resolution` to a structured field with proper validation. Until then, it lives happily in the unstructured metadata, consuming minimal resources and requiring no maintenance.

## 2.5 Type Conversion Disasters: A Field Guide

Every type conversion failure I've witnessed follows one of five patterns. Learn them, and you'll recognize the disaster before it ships.

### The ZIP Code Incident

The fundamental error: treating identifiers as quantities.

When you load a ZIP code column and let pandas infer the type, it sees five digits and thinks "integer." Mathematically, 90210 is now a number you can add, subtract, multiply, and divide. The model learns that Beverly Hills is 9.02 times "more" than Anchorage (99501). More what? Doesn't matter. The math works, so the model uses it.

The fix requires understanding what ZIP codes actually ARE: categorical labels that happen to be written with digits. They have no quantitative relationship to each other. 90210 isn't "bigger" than 10001 in any meaningful sense.

**The prevention:** Any column containing identifiers (ZIP codes, phone numbers, SSNs, product SKUs, user IDs) must be explicitly cast to string or category type BEFORE any analysis. Never trust type inference on columns with "id," "code," "zip," or "sku" in the name.

### The Boolean That Wasn't

Boolean values seem simple: true or false, 1 or 0. In practice, boolean columns can be represented six different ways (or MORE) in a single dataset. For example, the string "false", the integer 0, the actual boolean False, the string "N", NULL, and empty string could ALL mean "this is a no from me."

When you cast these to boolean in most languages, you get chaos. The string "false" evaluates to True because it's a non-empty string. The string "N" is also True. Only 0, False, None, and empty string give you False.

Your fraud detection model just learned that "is_fraud = false" means definitely fraud.

**The prevention:** Avoid casting, particularly with limited categories, to ANYTHING without explicit mapping, and VERY sanes defaults. For boolean, for example, build a lookup table that handles every representation in your data: true/false, t/f, yes/no, y/n, 1/0, and their various capitalizations - do NOT rely on upstream to handle this (e.g. "don't worry, we're only going to get handed lower case"). Anything not in your lookup should raise an error, not silently convert. And check your data often on errors!

### The Date That Broke Everything

Dates are a special hell because the same string can mean completely different things. "01/02/2024" is January 2nd in the US and February 1st everywhere else. "2024-01-15" is unambiguous ISO 8601, but it lives alongside "Jan 15, 2024", "20240115", Unix timestamps, Excel serial dates, and my personal favorite: "Monday."

The failure mode: your parser tries to be helpful by guessing formats. It guesses wrong on 3% of records. Those 3% silently become NULL or, worse, get parsed as the WRONG date. You don't notice until someone asks why all your European customers appear to have signed up on impossible dates.

**The prevention:** Know your date formats before you parse. If data comes from multiple sources, normalize at ingestion time with explicit format strings. Never use flexible parsing ("infer the format") on production data. When you see a date like "01/02/03", stop and figure out what it means before proceeding.

### The Phone Number Massacre

Phone numbers look like numbers. They're not. They're strings with specific formatting conventions that vary by country, carrier, and the mood of whoever entered the data.

When a type inference engine sees "555-0123", it helpfully computes 555 minus 123 and gives you 432. "555.012.3456" becomes 555.0123456 (a float!). "+1-555-012-3456" becomes 1 (everything after the first non-numeric character is discarded). 

Your customer contact list is now unusable.

**The prevention:** Phone numbers are ALWAYS strings. Period. If you need to validate or normalize them, use a purpose-built library (like `phonenumbers` in Python) that understands international formats. Never let numeric type inference touch a phone column.

### The Category Explosion

This one's subtle. You have a "category" column with clean categorical data, so you one-hot encode it for your model. Except the data isn't clean. It contains "Electronics", "electronics", "ELECTRONICS", "Electronic", "Electroncs" (typo), and " Electronics" (leading space).

After encoding, you have six separate binary columns for what should be ONE category. Your model learns that products with leading spaces in their category are completely different from products without. It's fitting noise.

**The prevention:** Normalize categorical values BEFORE encoding. Lowercase everything, strip whitespace, fix common typos, and map variations to canonical values. If you end up with more categories than you expected, investigate before proceeding.

## 2.6 The Type Safety Checklist

Run this audit before EVERY model training. Not some of the time. Every. Single. Time.

### Step 1: Hunt for Misclassified Identifiers

Look at every column currently typed as numeric (int, float). For each one, ask: does this column represent a QUANTITY or a LABEL?

Identifiers masquerading as numbers are the most common type failure. Check for columns with names containing "id," "code," "zip," "phone," "ssn," "isbn," or "sku." If a numeric column has one of these keywords, it's almost certainly wrong.

Also check the uniqueness ratio. If more than 95% of values are unique, you're probably looking at an identifier, not a measurement. Real measurements cluster; identifiers don't.

### Step 2: Find Hidden Numbers in String Columns

Examine every column typed as string/object. Try converting each to numeric. If the conversion succeeds without errors, you've found a number hiding as text.

This happens constantly with data imported from CSVs or JSON where everything comes in as strings. Revenue figures, quantities, prices - all sitting there as text, unable to be aggregated or compared.

### Step 3: Identify Dates Stored as Strings

Scan your string columns for date patterns. Look for formats like "2024-01-15", "01/15/2024", or "Jan 15, 2024". If you find them, those columns need to be converted to proper datetime types.

Dates stored as strings can't be sorted chronologically, can't have durations calculated, and will silently fail any time-based analysis.

### Step 4: Expose Fake Booleans

Check string columns for boolean indicators: "true/false", "yes/no", "y/n", "1/0", "t/f". If a column contains ONLY these values, it should be a proper boolean type, not a string.

Remember: the string "false" evaluates to True in a boolean context. This single fact has caused more fraud detection failures than any sophisticated attack.

### Step 5: Validate Value Ranges

For every numeric column, check that values fall within physically possible ranges:

- Ages should be 0-120, not -5 or 999
- Percentages should be 0-100 (or 0-1), not 150
- Prices should be positive
- Dates should be within your business's existence
- Counts should be non-negative integers

Sentinel values (999, -1, 9999-12-31) hiding in your data will corrupt every statistical measure.

### Step 6: Check Categorical Cardinality

For categorical columns, count the unique values. If you expect 5 categories and find 50, something's wrong - probably inconsistent capitalization, leading/trailing spaces, or typos creating phantom categories.

### The Conversion Strategy

When you find type problems, fix them with intention, not automation. Each type requires a different approach:

**Categorical conversions** (IDs, ZIP codes, SKUs): Always convert through string first to preserve leading zeros and formatting, then to category type for memory efficiency.

**Numeric conversions** (prices, quantities, measurements): Strip non-numeric characters before conversion. Use error handling that surfaces bad data as NULL rather than crashing - you need to SEE what didn't convert.

**Datetime conversions** (dates, timestamps): Try explicit format strings in order of likelihood for your data source. Only fall back to flexible parsing after explicit formats fail, and log everything that required inference.

**Boolean conversions** (flags, indicators): Build an explicit mapping for every representation in your data. Anything not in your mapping should error, not silently convert to a default.

### Required Reading

Before you convince yourself that your edge cases are handled, please read:
- ["Falsehoods programmers believe about names"](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)
- ["Falsehoods programmers believe about time"](https://gist.github.com/timvisee/fcda9bbdff88d45cc9061606b4b923ca)

Your confidence will be appropriately destroyed.

## Quick Wins Box: Type Fixes That Save Your Sanity

**1. Never Trust File Extensions (2 minutes)**

File extensions lie. That ".csv" might be Excel, that ".json" might be malformed XML, and that ".xlsx" might be a ZIP file someone renamed.

Before you try to parse anything, read the first 1024 bytes and check the actual file signature. ZIP files (including Excel) start with "PK". PDFs start with "%PDF". PNGs have a distinctive binary header. Parquet files start with "PAR1".

If the file is text, look for structural clues: curly braces and quotes suggest JSON; commas and newlines suggest CSV; angle brackets suggest XML.

This takes two minutes and saves hours of debugging "why won't this CSV parse" when the file was never a CSV to begin with.

**2. Document Your Schema Before You Forget (10 minutes)**

The moment you understand a dataset, write it down. For every column, record: the data type, the number of unique values, the null count and percentage, and either sample values (for strings) or the min/max range (for numbers).

This documentation doesn't need to be fancy. A markdown file is fine. The point is that six months from now, when someone asks "what does the 'status_code' column contain?", you'll have an answer that doesn't require re-analyzing the entire dataset.

The ten minutes you spend documenting now saves hours of archaeology later. And use git/source control/versioning to record your discovery! `json_schema` is your friend.

## Your Homework

### Exercise 1: The Type Audit (30 minutes)

Take a dataset you're currently working with and examine every column. For each one, answer three questions:

1. **What type does your tool think this column is?** (integer, float, string, datetime, etc.)
2. **What's the uniqueness ratio?** Divide unique values by total rows. Above 95% suggests an identifier; below 10% suggests a category.
3. **What do actual sample values look like?** Pull three random non-null values and look at them with human eyes.

Now compare what the tool inferred against what the data actually IS. Count the mismatches.

If you find more than three columns with wrong types, you have real work to do before any modeling. If you find zero, you're either lying or you haven't looked hard enough. Every dataset I've ever audited had at least one type problem.

### Exercise 2: The Structured vs Raw Decision (20 minutes)

Pick one unstructured data source in your organization. Answer these questions:
1. What structured fields do we actually USE from this data today?
2. What would it cost to extract those fields reliably?
3. What are we keeping "just in case" that we've never actually used?

If the answers are "not much," "a lot," and "most of it," consider whether you're over-engineering.

### Exercise 3: The Four Horsemen Hunt (45 minutes)

Load a dataset you trust. Try to find examples of each horseman:
1. **Structural lies**: Files that don't parse cleanly
2. **Type traps**: Numbers that shouldn't be numbers
3. **Semantic sinkholes**: Valid types with impossible values
4. **Schema mirages**: Overly rigid structures that lose information

I guarantee you'll find at least two.

## Parting Thoughts

Data types seem boring until they're not. The difference between a ZIP code as a number and a ZIP code as a category is the difference between a working model and a $50M toilet paper order.

In Chapter 3, we'll dive into file formats - JSON's lies, Parquet's promises, and Arrow's revolution. We'll explore why that "simple CSV" is anything but, and why the format you store data in matters almost as much as the data itself.

Until then, go audit your data types. Yes, right now. I promise you'll find something that makes you question everything.
