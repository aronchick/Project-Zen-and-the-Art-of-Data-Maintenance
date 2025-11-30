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

Mathematically sound. Practically insane.

Because the model had been performing so well on validation data, no one thought to triple-check. One week later, they discovered they'd auto-ordered 50,000 units of toilet paper for their jewelry department.

What's the failure here? In many ways, this is a "happy case" - at least the pipeline didn't crash, the system didn't error out, and nobody woke up at 3 AM to figure out why the website was down.

On the other hand, this is the worst of all possible worlds. The error went through ALL the systems with no warnings. You're going to spend hours (days?) debugging it because you're not getting ANY signal about what to do next.

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

Data doesn't just fail; it actively conspires against you. The most dangerous data isn't the obviously unstructured mess - it's the data that *pretends* to be structured, luring your code into a false sense of security before it detonates.

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

Your beautiful, strict parser ends up being so brittle that it either breaks on valid inputs or, worse, throws away most of your data.

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

This principle challenges the common instinct to immediately structure and normalize all incoming data. Instead, it advocates for a pragmatic approach: only invest effort in structuring data when you have a clear, immediate use case.

### The Hospital Records Lesson

A hospital system decided to "modernize" by converting 20 years of medical records into structured data. Budget: $2M. Timeline: 6 months.

**Month 1**: "We'll use OCR!"
(Narrator: They would not use OCR.)

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

## 2.5 Type Conversion Disasters: A Cookbook of What Not to Do

These are real examples from production systems. I've changed the names to protect the guilty.

### The ZIP Code Incident (Opening Story Expanded)

```python
# What happened
df['zip_code'] = pd.to_numeric(df['zip_code'])
model.train(df)

# What the model learned
# Beverly Hills (90210) is 9.02 times "more" than Anchorage (99501)
# Therefore people in 90210 must be 9.02x richer/riskier/something

# The fix that took 3 weeks to deploy
df['zip_code'] = df['zip_code'].astype(str)
# Or better:
df = pd.get_dummies(df, columns=['zip_code'])
```

### The Boolean That Wasn't

```python
# The data
{
  "is_fraud": "false",  # String "false"
  "is_verified": 0,      # Integer 0
  "is_active": False,    # Boolean False
  "is_deleted": "N",     # String "N"
  "is_flagged": None,    # NULL
  "is_suspicious": ""    # Empty string
}

# What pandas did
df.astype(bool)
# "false" → True (non-empty string)
# 0 → False (correct!)
# False → False (correct!)
# "N" → True (non-empty string)
# None → False (maybe correct?)
# "" → False (maybe correct?)

# 33% correct. Not great for fraud detection.
```

### The Date That Broke Everything

```python
# Actual production dates I've seen
dates = [
    "2024-01-15",           # ISO 8601 (good!)
    "01/15/2024",           # US format
    "15/01/2024",           # EU format  
    "Jan 15, 2024",         # Human readable
    "20240115",             # Compact
    "1705334400",           # Unix timestamp
    "44941",                # Excel serial date
    "2024-01-15T00:00:00Z", # ISO with time
    "Monday",               # Just... Monday
    "Yesterday",            # Helpful
    "TBD",                  # Thanks
    "[[ERROR]]",            # At least it's honest
]

# What the parser did: Gave up and returned NaT for everything
```

### The Phone Number Massacre

```python
# Original data
phones = [
    "555-0123",
    "5550124",
    "(555) 012-3456",
    "+1-555-012-3456",
    "555.012.3456",
    "Call me at five five five, zero one two three"
]

# After "helpful" type inference
phones_inferred = [
    -123.0,  # Subtraction!
    5550124.0,  # Integer as float!
    NaN,  # Gave up!
    1.0,  # Just... 1?
    555.0123456,  # Division!
    NaN  # Gave up again!
]
```

### The Category Explosion

```python
# Actual product categories from an e-commerce site
categories_before = [
    "Electronics",
    "electronics",
    "ELECTRONICS",
    "Electronic",
    "Electroncs",  # Typo
    "Elec.",
    "elect",
    "電子製品",  # Japanese
    "Electronics & Gadgets",
    "Electronics/Computers",
    " Electronics",  # Leading space
    "Electronics ",  # Trailing space
]

# After one-hot encoding without cleaning
# Created 12 columns for the same category
# Model learned that " Electronics" and "Electronics " are completely different
# Accuracy: 62%

# After cleaning
categories_clean = ["electronics"] * 12
# Accuracy: 84%
```

## 2.6 The Type Safety Checklist

Run this before EVERY model training. Seriously. EVERY. TIME.

```python
def validate_data_types(df):
    """
    Type safety audit - catches the disasters before they catch you.
    """
    
    issues = []
    
    # Check for numeric columns that shouldn't be
    for col in df.select_dtypes(include=['number']).columns:
        if any(keyword in col.lower() for keyword in 
               ['id', 'zip', 'phone', 'ssn', 'isbn', 'code', 'sku']):
            issues.append(f"⚠️  {col} is numeric but looks like an identifier")
    
    # Check for object columns that should be numeric
    for col in df.select_dtypes(include=['object']).columns:
        try:
            pd.to_numeric(df[col])
            issues.append(f"🔢 {col} could be numeric")
        except:
            pass  # It's actually text, good
    
    # Check for suspiciously uniform distributions (probably IDs)
    for col in df.columns:
        if df[col].nunique() / len(df) > 0.95:
            issues.append(f"🔑 {col} might be an ID (95% unique values)")
    
    # Check for date-like strings
    for col in df.select_dtypes(include=['object']).columns:
        if df[col].astype(str).str.match(r'\d{4}-\d{2}-\d{2}').any():
            issues.append(f"📅 {col} looks like dates stored as strings")
    
    # Check for boolean-like strings
    for col in df.select_dtypes(include=['object']).columns:
        unique_lower = df[col].dropna().astype(str).str.lower().unique()
        bool_indicators = {'true', 'false', 'yes', 'no', 'y', 'n', '0', '1'}
        if set(unique_lower).issubset(bool_indicators):
            issues.append(f"✓/✗ {col} looks like booleans stored as strings")
    
    return issues

# Run it
issues = validate_data_types(your_dataframe)
if issues:
    print("FIX THESE BEFORE TRAINING:")
    for issue in issues:
        print(f"  {issue}")
```

### The Universal Type Converter

```python
def safe_type_convert(series, target_type):
    """
    Converts types without crying.
    """
    if target_type == 'category':
        # For IDs, ZIP codes, etc.
        return series.astype(str).astype('category')
    
    elif target_type == 'numeric':
        # Remove non-numeric characters first
        cleaned = series.astype(str).str.replace(r'[^0-9.-]', '', regex=True)
        return pd.to_numeric(cleaned, errors='coerce')
    
    elif target_type == 'datetime':
        # Try multiple formats
        for fmt in ['%Y-%m-%d', '%m/%d/%Y', '%d/%m/%Y', '%Y%m%d']:
            try:
                return pd.to_datetime(series, format=fmt)
            except:
                continue
        return pd.to_datetime(series, errors='coerce')
    
    elif target_type == 'boolean':
        # Handle various boolean representations
        mapping = {
            'true': True, 'false': False,
            't': True, 'f': False,
            'yes': True, 'no': False,
            'y': True, 'n': False,
            '1': True, '0': False,
            1: True, 0: False
        }
        return series.astype(str).str.lower().map(mapping)
    
    return series  # Give up gracefully
```

### Required Reading

Before you convince yourself that your edge cases are handled, please read:
- ["Falsehoods programmers believe about names"](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)
- ["Falsehoods programmers believe about time"](https://gist.github.com/timvisee/fcda9bbdff88d45cc9061606b4b923ca)

Your confidence will be appropriately destroyed.

## Quick Wins Box: Type Fixes That Save Your Sanity

**1. The Format Detector (2 minutes)**
```python
def detect_format(file_path):
    """Figures out what you're actually dealing with."""
    with open(file_path, 'rb') as f:
        header = f.read(1024)
    
    if header.startswith(b'PK'): return 'zip_or_excel'
    elif header.startswith(b'%PDF'): return 'pdf'
    elif header.startswith(b'\x89PNG'): return 'png'
    elif header.startswith(b'\xff\xd8\xff'): return 'jpeg'
    elif b'<?xml' in header: return 'xml'
    elif header.startswith(b'PAR1'): return 'parquet'
    
    try:
        text = header.decode('utf-8')
        if '{' in text and '"' in text: return 'probably_json'
        elif ',' in text and '\n' in text: return 'probably_csv'
    except:
        return 'binary_mystery'
    
    return 'complete_mystery'
```

**2. The Schema Documenter (10 minutes)**
```python
def document_schema(df, output_file='schema.md'):
    """Creates documentation that future-you will thank present-you for."""
    with open(output_file, 'w') as f:
        f.write(f"# Data Schema Documentation\n\n")
        f.write(f"Generated: {datetime.now()}\n\n")
        
        for col in df.columns:
            f.write(f"## {col}\n")
            f.write(f"- Type: {df[col].dtype}\n")
            f.write(f"- Unique values: {df[col].nunique()}\n")
            f.write(f"- Nulls: {df[col].isna().sum()} ({df[col].isna().mean():.1%})\n")
            
            if df[col].dtype == 'object':
                f.write(f"- Sample values: {df[col].dropna().head(3).tolist()}\n")
            else:
                f.write(f"- Range: [{df[col].min()} - {df[col].max()}]\n")
            f.write("\n")
    
    print(f"Schema documented in {output_file}")
```

## Your Homework

### Exercise 1: The Type Audit (30 minutes)

Take your current dataset and run:

```python
for col in df.columns:
    print(f"\n{col}:")
    print(f"  Pandas type: {df[col].dtype}")
    print(f"  Unique ratio: {df[col].nunique() / len(df):.2%}")
    print(f"  Sample values: {df[col].dropna().sample(min(3, len(df[col].dropna()))).tolist()}")
```

Count how many columns have the wrong type. If it's more than 3, you have work to do. If it's 0, you're lying.

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

---

*P.S. - Your Boolean column contains the string "false". That evaluates to True in most languages. Sleep well.*

---
