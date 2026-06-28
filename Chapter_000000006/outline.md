# Chapter 6: Exploratory Data Analysis — The Art of Investigation (OUTLINE)

> Working outline, not prose. Follows the book's chapter structure: disaster cold-open →
> practical content → Quick Wins → Homework → P.S. Voice: war stories, real numbers.

## Cold open (disaster story with real numbers)
- A model shipped on data nobody actually *looked* at. Candidate story: a distribution that
  shifted silently (e.g., a sensor/units change, or a join that fanned out rows), passed every
  unit test, and cost $X in bad predictions before someone finally plotted the column.
- Thesis: EDA isn't a phase you skip when you're busy. It's the cheapest insurance you'll ever
  buy. The 30 minutes of looking you skip costs weeks of debugging downstream.

## 6.1 The Philosophy & Methodology of EDA
- EDA as investigation, not decoration. Tukey framing without the reverence.
- The mindset: assume the data is lying until it proves otherwise.
- Confirmatory vs. exploratory — why jumping to modeling skips the only step that's free.

## 6.2 Profiling Before Plotting
- Shape, dtypes, null counts, cardinality, ranges — the 60-second triage every dataset gets.
- Automated profiling (ydata-profiling/pandas-profiling, Polars) — useful, and where it lies.
- The "this column is 98% one value" and "this ID isn't unique" gotchas.

## 6.3 Distributions & The Lies of Summary Stats
- Anscombe's quartet / Datasaurus — same mean & std, wildly different reality. Look, don't trust.
- Skew, multimodality, and why mean/median divergence is a flag, not a footnote.
- Log scales and when a "normal" column is three populations stacked.

## 6.4 Relationships, Leakage & Anomalies
- Correlation heatmaps and their traps; spurious vs. real.
- **Target leakage caught in EDA** — the feature that's "too good." (bridge to Ch 12/14)
- Anomaly/outlier *spotting* here (full treatment in Ch 11): visual + simple statistical.
- Missingness patterns as signal (bridge to Ch 10: MCAR/MAR/MNAR).

## 6.5 EDA at Scale (when the data won't fit)
- Sampling strategies that don't lie; stratified peeks.
- Polars / DuckDB / Arrow for profiling big data without melting your laptop (bridge to Ch 3).
- Approximate stats (sketches, t-digest) for billions of rows.

## 6.6 Documenting & Communicating Findings
- The EDA memo: what you found, what it means, what you changed because of it.
- Making EDA reproducible (notebooks that aren't write-only).
- Handing findings to stakeholders without a slide deck full of seaborn defaults.

## Quick Wins Box
- The 60-second dataset triage checklist (shape, dtypes, nulls, cardinality, dupes, ranges).
- Three plots to run on every dataset before you do anything else.
- One leakage check that pays for the whole chapter.

## Homework
- Exercise 1: Profile a real dataset cold; write the one-paragraph "what's wrong here."
- Exercise 2: Find one column whose summary stats lie; prove it with a plot.
- Exercise 3: Hunt for target leakage in a dataset you've already modeled. (uncomfortable.)

## P.S. (closing humor)
- Riff: "The data scientist who skips EDA is the surgeon who skips the X-ray because the
  patient seemed fine in the waiting room."

---
### Source threads to mine
- Ch 1 (data-centric philosophy), Ch 2 (types), Ch 5 (quality dimensions) all feed this.
- Forward links: Ch 10 (missing), Ch 11 (outliers), Ch 12/14 (leakage, features).
