# Chapter 5: Data Acquisition and Quality Frameworks — OUTLINE

## Or: Where Your Data Actually Comes From (And Why It's Already Lying to You)

---

## Opening Disaster Story (~500 words)

**The Zillow Catastrophe**: Zillow's iBuying algorithm that led to $881 million in losses and 25% workforce layoffs. The data acquisition strategy relied on:
- Third-party listing feeds with inconsistent update frequencies
- User-submitted "Zestimates" that homeowners gamed
- MLS data that lagged actual market conditions by weeks
- Synthetic confidence intervals that masked fundamental uncertainty

The postmortem revealed: they had excellent models trained on fundamentally unreliable data sources. No amount of model sophistication could overcome acquisition-layer garbage.

**Bridge from Chapter 4**: You now know what bad data costs. This chapter is about not acquiring bad data in the first place.

---

## 5.1 The Acquisition Hierarchy: Where Data Actually Comes From (~1,500 words)

### 5.1.1 First-Party Data: Your Own Exhaust
- **What it is**: Data you generate through your own operations
- **The goldmine**: Event logs, user interactions, transactional records
- **The lie**: "We have all this data!" (No, you have all this *raw exhaust* with no schema)
- **Real example**: Stripe's approach to turning payment metadata into ML features
- **The hidden cost**: Instrumentation debt (you can't analyze what you didn't log)

### 5.1.2 Second-Party Data: The Partnership Trap
- **What it is**: Direct data sharing agreements
- **The promise**: "Our partner will give us their customer data!"
- **The reality**: Schema mismatches, update cadence conflicts, liability nightmares
- **Real example**: Retail co-op data sharing failures
- **When it works**: Clear contracts, shared incentives, technical alignment

### 5.1.3 Third-Party Data: The Vendor Roulette
- **What it is**: Purchased data from aggregators
- **The pitch**: "We have 300 million consumer profiles!"
- **The reality**: 40% stale, 20% duplicates, 15% fabricated, ???% accurate
- **Real example**: Marketing data vendor quality audit results
- **Due diligence that actually works**: Sample-then-verify, not trust-then-discover

### 5.1.4 Scraped Data: The Legal Minefield
- **What it is**: Programmatically collected from public sources
- **The temptation**: "It's on the internet, it's public!"
- **The reality**: ToS violations, rate limiting, legal exposure (hiQ vs LinkedIn)
- **When it's legitimate**: Academic research, competitive intelligence (with guardrails)
- **Real example**: The Common Crawl vs. targeted scraping tradeoffs

### 5.1.5 Synthetic Data: The New Frontier
- **What it is**: Algorithmically generated data mimicking real distributions
- **The promise**: Infinite training data! Privacy-safe! No collection costs!
- **The danger**: You're training on your assumptions about reality
- **When it works**: Data augmentation, privacy preservation, edge case generation
- **When it fails**: Primary training data, novel distribution discovery
- **Real example**: Waymo's synthetic driving scenarios vs. Tesla's real-world approach

---

## 5.2 The Six Dimensions of Data Quality (~2,000 words)

**Framing**: These aren't academic categories—they're the six ways your data will betray you.

### 5.2.1 Accuracy: Is It True?
- **Definition**: Does the data reflect reality?
- **The measurement problem**: Ground truth is expensive
- **Sampling strategies**: When you can't verify everything
- **Real example**: Medical imaging labels—radiologist disagreement rates
- **Practical threshold**: What accuracy do you actually need? (hint: less than you think)

### 5.2.2 Completeness: Is It All There?
- **Definition**: Missing values, missing records, missing relationships
- **The three types of missing**: MCAR, MAR, MNAR (and why it matters)
- **The 80% lie**: "Our data is 80% complete!" (Which 20% is missing?)
- **Real example**: Survey response bias in customer satisfaction data
- **Practical approach**: Quantify missingness patterns, not just rates

### 5.2.3 Consistency: Does It Agree With Itself?
- **Definition**: Same entity, same representation everywhere
- **Cross-system hell**: Customer in CRM vs. billing vs. support vs. product
- **Temporal consistency**: Does yesterday's "active user" mean today's "active user"?
- **Real example**: The company with 47 different definitions of "customer"
- **Practical approach**: Canonical definitions, enforced at write time

### 5.2.4 Timeliness: Is It Fresh Enough?
- **Definition**: Data age relative to decision requirements
- **The latency spectrum**: Real-time → near-real-time → batch → historical
- **Over-engineering alert**: Not everything needs to be real-time
- **Real example**: Fraud detection (milliseconds) vs. quarterly reporting (weeks)
- **Practical approach**: Match freshness requirements to actual decision cadence

### 5.2.5 Validity: Does It Follow the Rules?
- **Definition**: Data conforms to defined formats and constraints
- **The schema lie**: "We have a schema!" (Is it enforced? Current? Complete?)
- **Constraint types**: Type, range, format, referential, business logic
- **Real example**: Phone numbers that pass regex but aren't real numbers
- **Practical approach**: Validation at ingestion, not discovery

### 5.2.6 Uniqueness: Is It Counted Once?
- **Definition**: No unintended duplicates
- **The join explosion**: How merges create phantom records
- **Entity resolution hell**: Is "John Smith" at "123 Main St" the same person?
- **Real example**: Marketing attribution with 3x overcounted conversions
- **Practical approach**: Deterministic IDs, probabilistic matching, and knowing which you're using

---

## 5.3 Building a Quality Assessment Framework (~1,500 words)

### 5.3.1 The Data Quality Scorecard
- **Why scorecards fail**: Vanity metrics that don't connect to outcomes
- **What actually works**: Tying quality dimensions to business impact
- **Template with teeth**: Scorecard that triggers action, not reports
- **Real example**: Netflix's data health scoring system

### 5.3.2 Automated Profiling: Know Your Data Before It Knows You
- **Tool landscape**: Great Expectations, Deequ, pandas-profiling, whylogs
- **What to profile**: Statistical distributions, cardinality, correlations, freshness
- **The profiling trap**: Reports nobody reads
- **Real example**: Setting up continuous profiling that triggers alerts
- **Cost-benefit**: When automated profiling pays off (hint: earlier than you think)

### 5.3.3 Quality Gates: Stop Bad Data at the Border
- **Philosophy**: Prevention > detection > remediation
- **Gate placement**: Source → ingestion → transformation → serving
- **Hard gates vs. soft gates**: When to block vs. when to flag
- **Real example**: Airbnb's data quality gates for ML features
- **The override problem**: When business pressure bypasses quality gates

### 5.3.4 Quality SLAs: Making Quality Measurable
- **What to SLA**: Freshness, completeness, accuracy bounds
- **Who owns what**: Data producers vs. data consumers vs. platform
- **Enforcement mechanisms**: Alerts, escalations, incident processes
- **Real example**: Uber's data SLA framework

---

## 5.4 Data Contracts: The Missing Link (~1,000 words)

### 5.4.1 What Data Contracts Actually Are
- **Definition**: Formal agreements between data producers and consumers
- **Why they matter**: Schema changes break things; contracts prevent surprises
- **The social contract problem**: Technical enforcement vs. organizational buy-in

### 5.4.2 Contract Components
- **Schema definition**: Fields, types, nullability, constraints
- **Semantic definition**: What the fields actually mean (the hard part)
- **Quality expectations**: Freshness, completeness, accuracy bounds
- **Ownership and support**: Who to call when it breaks

### 5.4.3 Implementation Approaches
- **Schema registries**: Confluent, AWS Glue, Protobuf
- **Contract testing**: Consumer-driven contract testing for data
- **Versioning strategies**: Breaking vs. non-breaking changes
- **Real example**: How Spotify manages data contracts across 200+ teams

---

## 5.5 Metadata: The Context That Makes Data Usable (~1,200 words)

### 5.5.1 The Metadata Crisis
- **The discovery problem**: "I know we have this data somewhere..."
- **The interpretation problem**: "What does 'status_code' mean?"
- **The trust problem**: "When was this last updated? Is it still valid?"

### 5.5.2 The Three Types of Metadata
- **Descriptive**: What is this data? (Names, descriptions, tags)
- **Structural**: How is it organized? (Schema, format, partitioning)
- **Administrative**: Who owns it? (Lineage, ownership, access, freshness)

### 5.5.3 Metadata Management Systems
- **Tool landscape**: DataHub, Apache Atlas, Amundsen, Atlan
- **The catalog trap**: Building a catalog nobody uses
- **What makes catalogs sticky**: Integration with workflow, not separate destination
- **Real example**: LinkedIn's DataHub adoption strategy

### 5.5.4 Automated Metadata Collection
- **Schema inference**: Good for structure, terrible for semantics
- **Lineage extraction**: Tracking data flow through transformations
- **Usage analytics**: What data is actually being used?
- **The human layer**: Why you still need manual curation

---

## 5.6 Data Versioning and Lineage (~1,200 words)

### 5.6.1 Why Versioning Matters
- **Reproducibility**: Can you rebuild yesterday's model?
- **Debugging**: When did the data change?
- **Compliance**: Audit trails and regulatory requirements
- **Experimentation**: A/B testing on different data versions

### 5.6.2 Versioning Approaches
- **Git for data**: DVC, LakeFS
- **Snapshot-based**: Delta Lake, Iceberg time travel
- **Immutable logs**: Event sourcing patterns
- **Tradeoffs**: Storage cost vs. flexibility vs. query performance

### 5.6.3 Lineage: Following the Breadcrumbs
- **What lineage captures**: Source → transformation → destination
- **Why lineage matters**: Impact analysis, debugging, compliance
- **Lineage granularity**: Table-level vs. column-level vs. row-level
- **Tool landscape**: Apache Atlas, OpenLineage, Marquez

### 5.6.4 Practical Implementation
- **Start small**: Begin with critical paths, not everything
- **Automate extraction**: Integrate with your orchestration tools
- **Make it queryable**: Lineage nobody can search is lineage nobody uses
- **Real example**: Tracking a data quality issue from report to source

---

## 5.7 The Acquisition Audit: A Practical Framework (~800 words)

### 5.7.1 Before You Acquire: Due Diligence Checklist
- **Source reliability**: Track record, update frequency, methodology
- **Schema stability**: How often does the format change?
- **Coverage and bias**: What's included? What's systematically missing?
- **Legal and compliance**: Rights to use, retention requirements, deletion obligations

### 5.7.2 During Acquisition: Ingestion Safeguards
- **Schema validation**: Reject or quarantine non-conforming records
- **Quality profiling**: Continuous monitoring against baseline
- **Duplicate detection**: Dedup at ingestion, not discovery
- **Lineage capture**: Tag with source, timestamp, batch ID

### 5.7.3 After Acquisition: Continuous Monitoring
- **Drift detection**: Distribution changes that signal problems
- **Freshness monitoring**: Is data arriving when expected?
- **Quality trending**: Are things getting better or worse?
- **Usage tracking**: Is anyone actually using this data?

---

## Quick Wins Box: What You Can Do Monday Morning

1. **Audit one data source** using the six quality dimensions (~2 hours)
2. **Set up basic profiling** with pandas-profiling or whylogs for one table (~1 hour)
3. **Document one data contract** between your team and an upstream provider (~3 hours)
4. **Create a metadata entry** for your most critical dataset (~1 hour)
5. **Run a lineage trace** for one report back to its sources (~2 hours)

---

## Chapter Summary

- Data acquisition strategy determines data quality ceiling—you can't clean your way out of fundamentally bad sources
- The six quality dimensions (accuracy, completeness, consistency, timeliness, validity, uniqueness) are how data betrays you
- Quality frameworks need teeth: gates that block, SLAs that alert, scorecards that trigger action
- Data contracts formalize the relationship between producers and consumers
- Metadata solves discovery, interpretation, and trust problems—but only if people use it
- Versioning and lineage enable reproducibility, debugging, and compliance

---

## Bridge to Chapter 6

You now know where data comes from and how to assess whether it's any good. But assessment requires investigation—you need to actually look at your data, understand its distributions, find its anomalies, and develop intuition about what's normal and what's broken.

That's exploratory data analysis: the art of asking your data questions before you start making demands. And it's harder than it sounds—because data is very good at lying when you ask the wrong questions.

---

**Target word count**: ~9,000-10,000 words
**Estimated time to write**: 6-8 hours
**Key differentiators from other books**:
- Opens with Zillow disaster, not abstract quality dimensions
- Treats acquisition source as a first-class decision, not assumed
- Data contracts as missing organizational link
- Practical "Monday morning" actions throughout
- Connects quality dimensions to business impact, not academic completeness

---

## Research/Examples Needed

- [ ] Zillow iBuying postmortem details (public filings, news coverage)
- [ ] Netflix data health scoring (if public)
- [ ] Spotify data contract approach (engineering blog)
- [ ] LinkedIn DataHub case study (open source docs)
- [ ] Uber data SLA framework (engineering blog)
- [ ] Airbnb quality gates (engineering blog)
- [ ] Current tool landscape for profiling, catalogs, lineage (2024-2025 state)
