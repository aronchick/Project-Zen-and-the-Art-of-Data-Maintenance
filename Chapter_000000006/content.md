# Chapter 6: Exploratory Data Analysis: The Art of Investigation

## Or: How to Ask Your Data Questions Before It Lies to Your Face

A demand-forecasting team I worked with at an energy company had a model that worked. For two years it had forecast next-day electricity demand across a few hundred thousand metered customers, feeding a day-ahead procurement system that bought power on the wholesale market. When the forecast was right, the company bought roughly what it needed. When it was wrong, it either over-procured and ate the loss on resale, or under-procured and bought the shortfall at punitive real-time prices. For two years the forecast had been right often enough that nobody thought about it. Green dashboard. Boring MAPE. Life was good.

Then the meter vendor changed. Not all at once—that would have been easy to catch. They did the responsible thing and ran a rolling migration: a few regions at a time, over about six weeks. The new meters reported consumption on a slightly different basis than the old ones, under the same field name, with the same plausible-looking numbers. The schema didn't change. Every unit test passed, because every value was a valid floating-point number in a sensible range. The ingestion pipeline didn't blink. The quality gates—and unlike most teams, this one actually *had* quality gates, null checks and range checks and the works—waved it right through.

What changed was the *shape*. The consumption column had been a single, well-behaved, vaguely log-normal hump. After the migration it was two: the old fleet sitting where it always had, the new fleet shifted over by a consistent factor. The mean moved about 3%, well inside normal seasonal wobble, so the one summary statistic anyone glanced at looked unremarkable. The standard deviation nearly doubled—but who looks at standard deviation? Nobody looks at standard deviation.

Models almost never crash; that would be merciful, the same lesson the pipeline in Chapter 1 taught us on day 848. Instead it quietly got worse, region by region, as the migration rolled out. And as procurement drifted, the imbalance charges crept up. And when it was finally recognized as something that looked sticky enough to pay attention to, it took three weeks of a very smart team staring at model internals—retraining, tweaking features, suspecting the optimizer, suspecting the market—before someone finally, almost as an afterthought, pulled up a histogram of the raw input.

Two humps. Right there.

Once identified, the fix was easy: track down the vendor change, normalize the units, backfill. But the damage was done, and $2.3 million was out the door. The maddening part is that the disaster was *visible the entire time*, in plain sight, waiting for anyone to draw one picture. Nobody did, because the model was in production, and things in production are assumed fine until proven otherwise.

That assumption is the single most expensive habit in machine learning.

Exploratory data analysis is the discipline of *thoughtfully* refusing that assumption—and doing it as a habit, not as a thing you do once at kickoff and then skip forever because you're busy shipping. It is the cheapest insurance you will ever buy: thirty minutes of looking, traded against three weeks of debugging and a seven-figure hole. This chapter is about how to actually look—and, more importantly, *where* to look, in the places data has learned to hide.

---

## 6.1 The Philosophy and Methodology of EDA

John Tukey coined "exploratory data analysis" in 1977, and the term has since been buried under a half-century of reverent textbook treatment that mostly misses the point. EDA is not a checklist of plots or a boilerplate `df.describe()` cell at the top of every notebook that everyone runs and nobody reads. It is an *adversarial* activity—the data equivalent of a security red team. You are not admiring your data; you are interrogating a suspect who has every incentive to lie to you.

If you think of EDA as decoration—pretty charts to put in the project kickoff deck—you'll generate a dozen seaborn defaults, nod at them, and move on. If you think of EDA as investigation, you'll ask: *what would this data look like if it were broken in a way I haven't noticed yet, and have I actually ruled that out?*

The working assumption is simple and a little paranoid: **the data is lying until it proves otherwise.** It doesn't lie because it's malicious. It lies because it was created somewhere else, by someone else, for some other purpose, under assumptions nobody wrote down—the accumulated exhaust of a dozen upstream systems that have no idea you exist or what you're trying to do (Chapter 5 walked through where it all comes from). And every one of those upstream systems brings its own bugs, migrations, timezone confusions, and "temporary" hacks that became permanent. By the time the data reaches you, it has been through more hands than a dollar bill, and it shows.

There's a useful distinction here between *confirmatory* and *exploratory* analysis. Confirmatory analysis is when you have a hypothesis and you test it: "I think weekend demand is lower; let me check." Exploratory analysis is when you have no hypothesis and you let the data surprise you: "What's *in* here? What's weird? What doesn't belong?" Most people skip straight from "I have data" to "I'm training a model," which is a confirmatory act (the hypothesis being "this data can predict the target") performed without ever doing the exploratory work that would tell you whether the hypothesis is even sane.

The irony of the whole thing is that EDA is the only step in the entire pipeline that is essentially *free*—if you're curious and have zero budget, you can still do it! It needs no labeled data, no GPUs, no model, no infrastructure. It needs a laptop and your attention. The return on that afternoon is the highest in the business, and it is the step people are most likely to skip.

---

## 6.2 Profiling Before Plotting

Before you draw a single chart, you do triage. Triage is the sixty-second pass that tells you whether you're looking at a flesh wound or an arterial bleed, and it's almost entirely mechanical. Shape, types, nulls, cardinality, ranges, duplicates. Just the basics that give you the landscape you are playing on.

A routine process might look something like this:

```python
df.shape                      # how many rows, how many columns — is this even what I expect?
df.dtypes                     # what does the machine THINK each column is?
df.isna().mean().sort_values()  # null rate per column, worst last
df.nunique().sort_values()    # cardinality — the 1s and the all-uniques are both suspicious
df.describe(include='all')    # ranges, and the first place numbers look insane
df.duplicated().sum()         # how many rows are literally repeated?
```

While these are pretty simple, I have found these six lines to give quite a bit of tell and to be quite reusable. The first thing YOU should do is form your own, for your datasets and your business. Build that into a repeatable runbook, and run them before you do anything else. Read them like a detective, because each one has a specific tell, and whether or not you decide to use mine, you should have a line or two that gets you up to speed on each of the below.

**Shape**: What is the actual shape of your data? You expected 4.2 million rows and got 12.6 million? You probably have a join explosion (more on that in a moment). You expected 200 columns and got 47? Somebody's export silently truncated. The number that's wrong by a suspicious *multiple*—exactly 2x, exactly 3x—is a join fanout until proven innocent.

**Dtypes**: How are your objects being represented to the computer? Your ZIP codes are stored as `int64`, which means `02134` is now `2134` and a Boston address just teleported. Your dates are stored as `object`, which means they're strings and every date operation you do is going to be a special kind of hell. Your "amount" column is a string because three rows somewhere contain `"N/A"`. The machine's opinion of your types is a confession of every upstream sin.

**Null rates**: Are you seeing more holes in your data than you would be expecting? These are not interesting in aggregate; they're interesting in *pattern*. A column that's 4% null is a shrug. A column that's 4% null where the nulls are all concentrated in your highest-value customers is a catastrophe doing a convincing impression of a shrug—and triage won't tell you that, but it tells you *where to look*, which is the point. (The full taxonomy of missingness—MCAR, MAR, MNAR—is Chapter 10's job; right now you're just noticing that the holes exist.)

**Cardinality**: Is your data falling into the distribution patterns you expect? Both extremes can be a sign to investigate further. A column with cardinality 1 is a column that contains exactly one value—it's dead weight, and worse, it's often the fingerprint of a filter you didn't know was applied ("oh, this extract is only the US region"). A column with cardinality equal to the row count is either a genuine unique key or, very often, *not* the unique key you think it is. Which brings us to the most common gotcha in this entire section: **the ID that isn't unique.** You assume `customer_id` is one row per customer. You build a pipeline on that assumption. You're wrong—there are 1.03 IDs per row because of a historical merge—and now every aggregation you do is subtly off. Check it. `df.customer_id.is_unique` is one line. The absence of that one line has cost more than I'd care to total up.

The last two lines - descriptions and duplicates - are much more specific to your business, and fall into automated profiling. Tools like ydata-profiling (the library formerly known as pandas-profiling), or the lightweight `whylogs` we met in Chapter 5, will generate a gorgeous HTML report with every distribution, every correlation, every null pattern, all of it. Use them—they're genuinely a good accelerant. But understand where they lie. Automated profilers are *comprehensive*, which is exactly their weakness: a 500-column report that flags everything flags nothing, because no human reads 500 columns of warnings. They also choke or sample silently on large data, so the "distribution" you're admiring may be a distribution of the first 100,000 rows, which sorted by upload date, which means it's January and you're missing the entire year. The report is a starting point for your attention, not a substitute for it. The profiler tells you *what's there*. Only you can decide *what matters*, and deciding what matters is the entire job.

---

## 6.3 Distributions and the Lies of Summary Statistics

One thing that is the easiest to fall back on, and the easiest to get wrong, is the initial summary. **The summary statistic is where data goes to hide.** Mean, median, standard deviation, min, max—these are compression algorithms, and like all compression they throw information away. The whole game of EDA is knowing which discarded information was the part that mattered.

The statistician Frank Anscombe built a proof of exactly this danger. In 1973 he constructed four datasets—Anscombe's Quartet—that are statistically *identical*: same mean of x, same mean of y, same variance, same correlation, same regression line, down to two decimal places. By every number `df.describe()` would hand you, they are the same dataset. Plot them and they are four completely different worlds: one is a clean linear relationship, one is a smooth curve that a line badly misfits, one is a perfect line with a single wild outlier dragging it, and one is a vertical stack of identical x-values with one point off on its own deciding the entire slope.

The modern, gleefully petty version is the "Datasaurus Dozen"—a set of datasets with identical summary statistics, one of which, when plotted, is a dinosaur. The point of both of these is that you can have a column whose every number looks reasonable and whose actual shape is a literal cartoon, and you will never know unless you draw the picture.

So your job is to draw the picture. For every numeric column that matters, plot the histogram. It takes seconds and it catches the failure modes that summary stats are constitutionally incapable of catching:

**Multimodality**: How is the data distributed across the set? If you have two or more humps, you are almost never looking at one population; you're looking at two populations that got merged. This is the one that cost the energy company $2.3 million. Could be two vendors, two units, two regions, two definitions of the metric that changed on some date. Your mean will fall in the *valley between the humps*, describing a value that few of your actual records even have, which is effectively a "typical" customer who doesn't exist.

**Skew**: Do the mean and the median tell the same story? When they diverge, that gap is a *flag, not a footnote*—but what it means depends on the population the data came from. Income, file sizes, session lengths, basket values tend to be right-skewed, with a long tail of large values yanking the mean above the median. Feed that mean to a model or a stakeholder as "typical" and you've overstated the center for the bulk of your records. When mean and median disagree, the median is usually telling the truth and the mean is telling you there's a tail you need to deal with (and a transformation you'll meet in Chapter 12).

**Spikes**: Is there anything that is obviously non-continuous? A sharp vertical line at exactly zero, or at exactly 9999, or at -1, is almost never real data. It's a sentinel value—somebody's "missing," somebody's "unknown," somebody's "the sensor was offline so we wrote a zero," masquerading as a legitimate number. The pipeline in Chapter 1 that filled dead sensors with zeros produced exactly this: a histogram with a skyscraper at zero and a model that learned that broken machines are extremely healthy. Summary stats will fold that spike into the mean and hide it forever. The histogram shows it instantly.

A log scale is your friend here more often than you'd think. Plenty of "normal-looking" columns are three populations stacked across orders of magnitude—a few hundred small values, a few thousand medium ones, a handful of enormous ones—and on a linear axis the small ones smear into an indistinguishable lump against the wall while the giants blow out the range. Switch to a log x-axis and the structure separates out like oil and water. If a distribution looks like a single bar at the far left and nothing else, you're not looking at a single value; you're looking at a linear axis lying to you about a log-scale world.

None of this requires sophistication. It requires the discipline to type `df[col].hist()` before you trust `df[col].mean()`. The statistic is a summary of the picture. Never let the summary substitute for the picture.

---

## 6.4 Relationships, Leakage, and Anomalies

Once you trust your columns individually—and not before—you start looking at how they relate. This is where EDA stops being hygiene and starts being the thing that saves your model from itself.

The default move is a correlation heatmap, and the default move is a trap. Correlation heatmaps are seductive: a grid of pretty colors that *feels* like understanding. But Pearson correlation only measures *linear* association, so a perfect parabola—y rises, peaks, falls—can show a correlation near zero while being completely, deterministically related. Anscombe's curved dataset would look "uncorrelated-ish" in the wrong summary. Meanwhile, spurious correlations swarm: with a few hundred columns you will find pairs that correlate at 0.8 for no reason other than that you went looking through hundreds of pairs (the more comparisons you make, the more coincidences you manufacture). Use the heatmap to generate *questions*, never answers. Every strong correlation is a hypothesis to plot, not a fact to report.

**Target leakage:** does a feature carry information you won't actually have at prediction time—information that has, directly or indirectly, leaked from the very thing you're trying to predict? It is the most dangerous bug in machine learning precisely because it doesn't look like a bug. It looks like *success*.

EDA is where you catch it, and the tell is the feature that's *too good*. When you plot a single feature against your target and the relationship is suspiciously clean—when one column alone gets you to 0.99 AUC—you have almost certainly not discovered a brilliant predictor. You've discovered a leak. A classic example would be a `account_closed_date` feature in a churn model (of course it predicts churn—it *is* churn). Or a `total_paid` field in a default-prediction model that was computed *after* the loan resolved. Or a diagnosis code that gets entered only after the disease you're predicting has been confirmed. Or a code in a case-management field that gets populated *after* the fraud team had already flagged the account. Your model will pick up on these entries, and "predict" your results with far more knowledge than it will have when operating against an unknown data set. In production, in the BEST case, the fields are empty (since your model will fall back to other features). In the worst, they are filled with alternative values and performance is random or degrades.

Using EDA will help you ask a question up front: for every feature that shows a strong relationship with the target *would I actually know this value at the moment I need to make the prediction?* Plot the suspiciously-strong features first. (Chapters 12 and 14 deal with engineering and validating features properly; the point here is that the *first* place you catch leakage is by eyeballing feature-target relationships, long before you've built anything.)

Two more things you spot at this stage. **Anomalies and outliers**: you're not treating them yet—Chapter 11 owns the question of remove-versus-cap-versus-keep—but you're *spotting* them, because a scatter plot with three points sitting in deep space tells you something about either your data collection or your assumptions, and you want to know which before you model. A box plot or a simple z-score scan is enough at this stage. And **missingness as signal**: when you plot *where* the nulls fall, patterns emerge. If `income` is missing precisely for the customers who later churned, that missingness isn't noise to impute away—it's one of the most predictive things in your dataset, and how you handle it (Chapter 10) will make or break the model. Nulls are not always absence. Sometimes they're the loudest signal you have.

---

## 6.5 EDA When the Data Won't Fit on Your Laptop

Everything above assumes you can pull the data into memory and poke at it. What happens when "the data" is two billion rows in a warehouse and your laptop has 32 gigs of RAM and a strong sense of self-preservation?

Don't give up on EDA because the data is big—which is, perversely, exactly when people *do* skip it, right when the stakes are highest. The right answer is that big-data EDA is a solved problem; you just need the right tools and the discipline to sample honestly.

Start with sampling; most EDA questions don't need all two billion rows. A well-drawn sample of a million rows will show you the same histograms, the same modes, the same leakage, the same null patterns as the full dataset—*if* the sample is drawn honestly (the word "honestly" is doing a TON of work here). `LIMIT 1000000` is not a random sample; it's the first million rows, which are sorted by whatever the storage layer felt like, which is usually time, which means your "sample" is a single time slice with all the seasonal, regional, and version-specific structure that implies. The energy company's bug would have been *invisible* in a `LIMIT` sample taken before the vendor migration. Use real random sampling (`TABLESAMPLE`, `ORDER BY RANDOM() LIMIT n`, or your warehouse's reservoir sampler), and when you have rare classes that matter—fraud, churn, the 0.1% that's the whole point—use *stratified* sampling so the rare thing actually shows up in your sample instead of getting rounded away.

When you do need to touch all the rows—exact null rates, exact cardinality, exact min/max—push the computation to where the data lives instead of dragging the data to your laptop. This is where the columnar tooling from Chapter 3 earns its keep. **Polars** will profile datasets on a single machine that would make pandas fall over, because it's built on Arrow and actually uses all your cores. **DuckDB** lets you run SQL directly against Parquet files on disk or in object storage without loading anything—`SELECT count(*), count(DISTINCT id), avg(x) FROM 's3://bucket/*.parquet'` profiles a terabyte from your laptop because it only reads the columns and chunks it needs. For genuinely enormous data, Great Expectations and Deequ (both from Chapter 5) run quality profiles as distributed jobs.

And for the questions where an exact answer is overkill, approximate algorithms get you 99% of the insight for 1% of the cost. HyperLogLog estimates cardinality across billions of rows in kilobytes of memory—"how many distinct users?" answered without a giant `DISTINCT` sort. The t-digest estimates quantiles (medians, p99s) in a single streaming pass. These sketches are built into most warehouses (`approx_count_distinct`, `approx_quantile`), and for EDA—where you're trying to understand shape, not certify a financial report—approximate is almost always good enough. You don't need the exact median to notice the distribution has two humps.

The principle underneath all of it: scale changes your *tools*, not your *obligation*. Big data does not earn you a pass on looking. It just means you look with DuckDB instead of a histogram cell.

---

## 6.6 Documenting and Communicating What You Found

EDA that lives only in your head, or in a notebook you never reopen, was barely worth doing. The deliverable of exploration is not a feeling of familiarity—it's a written record that you and your team can act on, argue with, and come back to in six months when the model breaks and nobody remembers what "normal" looked like. Extra credit if you write run books so other folks on the team can execute when you're on vacation!

Write an EDA memo. It does not need to be long, and it should not be a data dump. It needs three things: **what you found, what it means, and what you changed because of it.** "The `consumption` column is bimodal after 2019-03; the second mode corresponds to the new meter vendor; I added a unit-normalization step and a vendor flag." That's a finding, an interpretation, and an action, in one sentence. Three sentences like that are worth more than a hundred auto-generated charts, because they encode *judgment*, and judgment is the thing that doesn't survive in a notebook full of `Out[47]:` cells.

Which brings up the reproducibility problem. Most EDA notebooks are write-only: cells run out of order, half of them error, the variable `df` has been redefined six times and nobody knows which version made the chart in cell 23. A notebook you can't rerun top-to-bottom is not analysis; it's a crime scene. Restart-and-run-all before you trust anything, keep the cells in an order that tells a story, and delete the dead exploration. Future-you, debugging this pipeline at 2 a.m. having forgotten everything (the same future-you Chapter 1 kept warning about), needs to be able to rerun your investigation and get your numbers.

Finally, communicating to stakeholders. The failure mode here is dumping twelve seaborn defaults with axis labels like `x` and `Unnamed: 0` into a deck and wondering why the room glazes over. Nobody outside your head cares about your correlation heatmap; they care about the *consequence*: "12% of our highest-value customers are missing the income field, which means our churn model is blind to exactly the people we most need to keep." That's a finding a VP can fund a fix for. One clean chart with a sentence of plain-English takeaway beats a wall of technically-correct visualizations every time. You did the investigation; now do the translation. The investigation is worthless if the person who can act on it doesn't understand what you found.

---

## Quick Wins: Do These Before You Train Anything

**The 60-second triage (60 sec).** On any dataset, before anything else, run the six lines: `shape`, `dtypes`, `isna().mean()`, `nunique()`, `describe()`, `duplicated().sum()`. Read each one like a detective. The wrong row count and the non-unique ID will save you a week apiece.

**The three plots that catch most disasters (5 min).** For your most important columns: (1) a histogram of every key numeric column—look for two humps and for skyscraper spikes at 0 / -1 / 9999; (2) a bar chart of every key categorical's value counts—look for the surprise 98%-one-value column and the `"N/A"` / `"null"` / `""` impostors hiding among real categories; (3) a missingness map (`df.isna()` heatmapped or `missingno`)—look for nulls that cluster instead of scattering.

**The one leakage check that pays for this whole chapter (10 min).** Fit a single feature against your target, one feature at a time, and rank them by how well each one *alone* predicts. Then take the top three and ask, for each: *would I actually have this value at prediction time?*

```python
# Dead-simple leakage smell test: which single columns predict the target too well?
from sklearn.metrics import roc_auc_score
for col in features:
    s = df[col].fillna(df[col].median()) if df[col].dtype != 'O' else df[col].astype('category').cat.codes
    try:
        auc = roc_auc_score(df[target], s)
        if max(auc, 1 - auc) > 0.95:          # one column, near-perfect → be very suspicious
            print(f"{col}: AUC {max(auc, 1-auc):.3f}  ← would I know this at prediction time?")
    except Exception:
        pass
```

Any single column over 0.95 is guilty until proven innocent. The feature that looks like magic is the feature that's reading the answer key.

---

## Your Homework (Yes, Still Homework)

### Exercise 1: Profile a Dataset Cold (Time: ~45 minutes)

Take a dataset you've *never* looked at—ideally one a colleague swears is clean. Run the 60-second triage and the three plots. Then write one paragraph: "Here's what's wrong with this data." I have never once seen this exercise come back empty. If yours does, you didn't look hard enough—go plot the histograms on a log scale.

### Exercise 2: Catch a Statistic in a Lie (Time: ~30 minutes)

Find one numeric column in your data where the mean and median differ by more than 20%. Plot its histogram. Identify *why* they diverge—skew, a second mode, or a sentinel-value spike. Write down what the mean was telling you and what the picture told you instead. Keep this one; it's the fastest way to make the lesson permanent.

### Exercise 3: Hunt Leakage in a Model You Already Shipped (Time: ~1 hour, uncomfortable)

Take a model already in production—yes, that one. Run the leakage smell test from Quick Wins against its training data. For every feature that scores above 0.9 alone, trace where the value actually comes from and *when* it gets populated. If you find a leak, you've just learned why your offline metrics were always a little better than reality. You're welcome, and I'm sorry.

---

## Bridge to Chapter 7

EDA tells you what your data *is*. Very often, the most important thing it tells you is that your labels are a mess—that the same thing is tagged three different ways, that "ground truth" is anything but, that the target you've been so carefully predicting was defined by a tired contractor at 4 p.m. on a Friday. Catching that is exploration's job. *Fixing* it is the next chapter's: data labeling and annotation, where we'll deal with the uncomfortable reality that the answer key you've been grading your models against is, itself, full of wrong answers.

---

*P.S. — A data scientist who skips EDA is a surgeon who skips the X-ray because the patient looked fine in the waiting room. The patient always looks fine in the waiting room. That's what waiting rooms are for. The histogram is the X-ray, it costs you thirty seconds, and unlike the surgeon, you don't even have to wash your hands first. Go plot something.*
