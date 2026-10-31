# Chapter 11: Outlier Detection and Treatment

## Or: The Number You Deleted Was the Business

A property and casualty insurer wrote about 120,000 claims a year across personal auto and homeowners, against roughly $740 million in annual incurred losses. They built a claim-severity model: at first notice of loss, predict what this claim will ultimately cost, so the reserve—the money the insurer sets aside to pay the claim—gets set correctly on day one instead of six months later when an adjuster finally gets to it. All great!

The model validated at an R² of 0.71 (R² is the share of the target's variation the model explains; 1.0 is perfect, 0 is no better than always guessing the average), which for claim severity is a strong number, so it went into production setting initial reserves.

Eleven months later the year-end actuarial review came back with a reserve strengthening charge of $26 million. Turns out the book had been systematically under-reserved all year, and the model had been confidently wrong in one direction the entire time.

The cause was two lines in a preprocessing notebook:

```python
cap = df["ultimate_loss"].quantile(0.99)
df["ultimate_loss"] = df["ultimate_loss"].clip(upper=cap)
```

Someone had winsorized the target at the 99th percentile: every claim above that line was pulled down to the 99th-percentile value. It's not that the person who wrote that was being lazy; they were solving a real problem, one they had investigated and correctly diagnosed. Claim severity is *extremely* heavy-tailed—most claims are small, but the rare enormous ones are frequent enough to carry a big share of the total dollars—and mean squared error, the average squared gap between prediction and actual that most regression models minimize, is dominated by the largest of those gaps, so a handful of catastrophic claims were jerking the training around so badly the model wouldn't converge cleanly (settle on a stable fit). Capping the target solved the problem, the loss curve got smooth, and the validation metric went up. Every signal a data scientist normally trusts said this was the right move.

In personal lines, the top 1% of claims by size accounted for about 34% of total loss dollars. A total house fire, a severe bodily injury with a lifetime care component, a liability verdict—these are rare and they are most of the money. Capping at the 99th percentile removed every example of the phenomenon that determines whether the company is profitable. The model was trained on a world in which the largest possible claim was $180,000, and then deployed into a world that regularly produces claims of $2 million.

So it did exactly what it was taught, and the model became superb at predicting fender-benders and water damage—the claims that are numerous, well-behaved, and financially irrelevant—and it had never once seen a claim above $180,000, so it reserved every severe claim at roughly the cap. Twelve hundred times a year it looked at a catastrophic loss and said "call it a hundred and eighty grand." The R² of 0.71 was measured on capped data, against a capped target, which is to say it was a precise measurement of the model's skill at a task nobody needed done.

They never deleted a row. They just erased the roughly $36 million a year that sat above the cap, and the model learned a world where that money did not exist.

---

The reflex that produced that disaster is the same reflex that produced Chapter 10's. Confronted with a value that makes the data awkward, reach for the standard move that makes it go away. Fill the null, clip the tail, get the core of the model right. In both cases the standard move is a bet about where the value came from, and in both cases the pipeline will run perfectly whether or not the bet was right.

First, the plain version: an outlier is a value that sits far from the rest of its column, or far from where the other columns say it should be—a $2 million claim in a table whose typical entry is $4,000, or a person recorded as 6'2" and 110 pounds. That is what every detector in this chapter finds. But an outlier is not a property of a number. It is a *claim about the process that generated the number*—specifically, the claim that this point came from somewhere your model shouldn't represent. That claim can be true, and frequently it is! But it's a claim about the world, it cannot be settled by a threshold, and the entire cost of getting it wrong lands in the tail, which is where a great many businesses keep their profit, their risk, and the thing they hired you to predict.

---

## 11.1 Defining Outliers: Statistical vs Domain-Based Approaches

The three-sigma rule is the most widely used piece of statistics in industry. Under a normal distribution (the bell curve), 99.7% of mass sits within three standard deviations of the mean—sigma, σ, being the standard deviation, the usual measure of how spread out a column is—so a point outside that range is rare: one in about 370. Flagging it as suspicious is reasonable. In code it's `np.abs(scipy.stats.zscore(x)) > 3`, where a z-score is how many standard deviations a value sits from the mean.

*Under a normal distribution.* This is a CRITICAL statement to get right. Claim sizes, incomes, transaction values, page views, session durations, file sizes, city populations, word frequencies, order quantities—all right-skewed (a long tail stretching off to the right), most of them roughly log-normal (normal only after you take the log) or worse. On a log-normal distribution, points five and six standard deviations above the mean are not anomalies, they happen every Tuesday. If you apply a three-sigma filter to a heavy-tailed variable and you are not removing errors, you are performing a distributional lobotomy and calling it data cleaning.

The mean and standard deviation have a second problem as well, which is that they are computed from the data including the outliers. A handful of values entered as 5,000,000 instead of 50 will inflate the standard deviation enough that the merely bad ones—the 50,000s—stop flagging at all. Statisticians call this *masking*, and it means the three-sigma rule is least reliable in exactly the situation you deployed it for.

The **IQR or Tukey fence** is better on both counts, because quartiles don't care how extreme the extreme values are. Q1 and Q3 are the 25th and 75th percentiles, the interquartile range (IQR) is the gap between them, and the fence flags anything below Q1 − 1.5×IQR or above Q3 + 1.5×IQR:

```python
q1, q3 = x.quantile([0.25, 0.75])
iqr = q3 - q1
flag = (x < q1 - 1.5 * iqr) | (x > q3 + 1.5 * iqr)
```

But it was designed with roughly symmetric distributions in mind, and on a strongly right-skewed variable it will flag a large slice of the upper tail by construction. Not because anything is wrong, but because the fence assumes a symmetry the data doesn't have. Run it on income data and watch it declare that several percent of the population is anomalous.

The **modified z-score** built on median absolute deviation is the best of the simple options. Take the median, take the median of the absolute deviations from that median, and scale: 0.6745 × (x − median) / MAD, flagging above about 3.5.

```python
from scipy.stats import median_abs_deviation
mad = median_abs_deviation(x)
modified_z = 0.6745 * (x - x.median()) / mad
flag = modified_z.abs() > 3.5
```

MAD has a breakdown point of 50%, meaning half your data would have to be corrupted before bad things start happening. Compare that to the standard deviation, where one bad value is enough. This is what statisticians mean by a *robust* statistic: one that a few wild values can't drag around. If you're going to use a threshold, use this one.

But all of this is procedure, and procedure is the *easy bit*. Every extreme value in your dataset is one of three things, and the statistics cannot answer that for you (sadly):

**It's an error.** A decimal point in the wrong place, a unit mix-up, a sensor reporting garbage, an undecoded sentinel like -999 or 9999 masquerading as a measurement (Chapter 2's problem, arriving late and expensive). Remove it or repair it.

**It's a different population.** A wholesale account sitting in a retail dataset. A bot in a table of human users. An internal test transaction that was never filtered out. This point is not wrong; it is correct data about a thing you weren't modeling. The right move is almost never deletion; it's segmentation, or a flag, or a separate model.

**It's a rare real event.** The house fire. The machine three hours from failure. The fraudulent transaction. The point is correct, it belongs to the population you're modeling, and it is very often the MOST IMPORTANT point. This is the outlier that you are being paid to predict.

Which gives you the one rule in this chapter that holds without qualification: **you may delete a data point only if you can name the mechanism that made it wrong.** "It's 4.2 sigma from the mean" is not enough. "The vendor changed the unit from liters to gallons in March and this batch wasn't converted" is a mechanism. If you can't name one, you are not removing an error—you are removing evidence, and you should keep the row and go find out what it is.

---

## 11.2 Univariate and Multivariate Detection Methods

Everything in 11.1 is univariate: one column at a time. Nearly all production outlier handling lives there and, sadly, it has a blind spot. A big one.

Consider a dataset of people. A height of 6'2" is unremarkable. A weight of 110 pounds is unremarkable. A 6'2" person weighing 110 pounds is medically extraordinary and almost certainly a data-entry error—and **no column-by-column check will ever find it**, because neither value is unusual on its own. The anomaly lives in the relationship, not in either number.

Your data is full of these. A customer with a two-week account age and a $40,000 lifetime spend. A ninety-minute session with one page view. A claim with a severe injury code and a $400 payout. Every one of them is invisible to a per-column scan, and every one of them is more interesting than anything a per-column scan will surface.

**Mahalanobis distance** is the classical answer: measure distance from the multivariate center in units of the covariance structure—covariance being how much two columns move together—so the metric knows that height and weight travel together. However, this assumes roughly elliptical structure, so it struggles with clusters and nonlinear relationships, AND it's computed from the sample mean and covariance matrix, which the outliers themselves corrupt—masking again, worse in higher dimensions. If you use it, you need more underneath it: a robust estimator such as Minimum Covariance Determinant, which fits the center and covariance on the tightest subset of the data and then measures everything against that.

```python
from sklearn.covariance import EmpiricalCovariance, MinCovDet
d2_naive = EmpiricalCovariance().fit(X).mahalanobis(X)      # outliers inflate it
d2_robust = MinCovDet(random_state=0).fit(X).mahalanobis(X)  # fit on the tightest half
```

Both return squared distances; scikit-learn's `EllipticEnvelope` wraps the robust version as a ready-made detector.

Unfortunately, this only gets you so far. In high dimensions—many columns, each one a dimension—distance stops discriminating. As you add dimensions, the distances between all pairs of points converge toward each other, and "far from the center" degrades into noise. Distance-based outlier detection works well in five dimensions, but works poorly in fifty, and is essentially decorative in five hundred—at least when those five hundred columns carry genuinely independent information, which wide real-world tables often don't. If you're operating in high dimensions, reduce first (Chapter 15) or use a method that doesn't depend on a global distance metric.

Two more categories also change what you look for:

**Contextual outliers** are only anomalous given their context. Forty degrees Fahrenheit is normal in January and alarming in July. A $5,000 charge is routine for one cardholder and a red flag for another. No global threshold captures this—you have to condition on time, entity, or segment, which means the detector needs the same point-in-time discipline as everything else in this book.

**Collective outliers** are anomalous as a group while every individual point looks fine. The classic is a sensor stream that goes perfectly flat: every reading is inside the normal range, and the *absence of variance* is the failure. A vibration sensor reporting exactly 0.42 for six straight hours has died, and every univariate check in the world will wave it through. Watch second-order properties—variance, autocorrelation (how much each reading resembles the one before it), the rate of change—not just values. A rolling standard deviation that hits zero, `s.rolling("6h").std() == 0` on a timestamp-indexed series, catches the dead sensor that every per-value check waves through.

---

## 11.3 Machine Learning-Based Anomaly Detection

When thresholds and distances run out, there's a family of methods that learn the shape of "normal" and score deviations from it.

**Isolation Forest** is the first stop, because its core idea works and it scales. Build random trees by picking a random feature and a random split point, over and over. Anomalies get isolated in very few splits—there's not much data around them, so a random cut separates them quickly—while normal points require many splits to fence off. The score is average path length. It's roughly linear in the number of samples, it needs no distance metric, and it handles moderate dimensionality far better than anything distance-based. Its `contamination` parameter is a trap: it asks you to declare what fraction of your data is anomalous, which is the thing you were trying to find out. Set it deliberately or leave it and threshold the raw scores yourself.

```python
from sklearn.ensemble import IsolationForest
iso = IsolationForest(random_state=0).fit(X)
scores = iso.score_samples(X)  # lower = more anomalous; threshold these yourself
labels = iso.predict(X)        # -1 anomaly, 1 normal, decided by contamination
```

**Local Outlier Factor** compares a point's local density to the density of its neighbors; `LocalOutlierFactor(n_neighbors=20).fit_predict(X)` returns −1 for the outliers. This catches something global methods structurally cannot: a point sitting in a sparse pocket that isn't far from the overall center. If your data has clusters of different densities—and real data does—LOF finds anomalies that Mahalanobis and Isolation Forest both miss.

**One-Class SVM** learns a boundary enclosing the normal region. Powerful and principled, sensitive to its kernel (the function that lets the boundary curve) and `nu` (roughly, the fraction of training points you're willing to leave outside the boundary), and it scales poorly. Useful when you have a clean sample of definitely-normal data to fit on: `OneClassSVM(nu=0.01).fit(X_normal)`, then `.predict(X_new)`.

**Autoencoder reconstruction error** is the deep-learning entry: train a network to compress and reconstruct normal data, then flag whatever it reconstructs badly. This works well on images, high-dimensional sensor arrays, and sequences, where the notion of "distance" is useless but the notion of "this doesn't look like the things I was trained on" is exactly right. It's a model you build in PyTorch or Keras, not a call you make. It carries all the operational baggage from Chapter 10—it's a second model, it drifts, it needs monitoring.

**DBSCAN**, a clustering method that groups points by density—a point joins a cluster if it has enough neighbors within a radius—gives you outliers as a side effect: points that never join a cluster are noise, `DBSCAN(eps=0.5, min_samples=5).fit_predict(X) == -1` on scaled features (`eps` is a distance in feature units), which is often what you wanted.

Now the structural limitation that applies to all of them. These are unsupervised methods—they learn from the inputs alone, with no labels telling them which rows were actually bad—which means they answer the question "is this statistically unusual?" But a BETTER question is "is this *wrong*, or is it the rare real thing I care about?" Nothing unsupervised can distinguish those, because the difference isn't in the data, it's in the mechanism. An anomaly detector produces a ranked list of things to look at. It's extremely important that you don't declare "that's the decision" because the moment you wire its output directly into `df.drop()` you've automated a judgment nobody made. To LOOK at! Not to DECIDE!

And the corollary that people miss constantly: **if you have labels, use them.** Labels are known answers for past rows: this transaction was fraud, this machine did fail. If you know which transactions were fraud, which machines failed, which claims blew up—train a supervised classifier, one that learns from those answers. It will beat every anomaly detector on this list on the anomalies your labels cover, by a lot. Anomaly detection is the technique you use when you don't have labels. Reaching for Isolation Forest on a labeled problem because the phrase "anomaly detection" sounds closer to the goal is a mistake that gets made constantly.

---

## 11.4 Treatment Strategies: Remove, Cap, Transform, or Keep

You found them. Now what? There are five real options, but choose carefully (see the intro to this chapter for why).

**Remove.** Justified only when you can name the mechanism—the rule from 11.1. Three disciplines make removal safe. Log every deletion with its reason, to a file, not to a comment. Count them as a percentage of your data; if you're dropping 5%, you have stopped cleaning and started reshaping. And remember that deleting a row in training does nothing to the world: whatever produced that value will produce it again next Tuesday, in production, to a model that has now never seen one.

**Cap or winsorize.** Bound the value without losing the row. `scipy.stats.mstats.winsorize(x, limits=(0.01, 0.01))` pulls the bottom and top 1% in to the 1st and 99th percentiles; the cold open's `clip(upper=cap)` is the one-sided pandas version. Legitimate when the quantity is genuinely bounded and the excess is definitionally impossible—for example, an age of 214, a body temperature of 900. But make sure you ask: **are you capping a feature or the target?** Capping a feature limits how much leverage one observation has over the fit—leverage being how far a single row with an extreme input can drag the fitted line toward itself—which is often right. Capping the target changes what the model is being asked to predict, and if the tail is where your economics live, you have just redefined the problem into one that doesn't matter. The insurer at the beginning of this chapter capped the target.

**Transform.** Apply a log (`np.log1p(x)`), a Box-Cox (`PowerTransformer(method="box-cox")`), a Yeo-Johnson (`PowerTransformer(method="yeo-johnson")`), or a quantile transform (`QuantileTransformer(output_distribution="normal")`, which forces the column onto a bell curve by rank). This is usually the right answer but it's underused because it feels less decisive than deletion. A log transform compresses a heavy tail into something a linear model can work with while preserving every row and every ordering—the largest claim is still the largest claim, it just no longer dominates the squared error. Plain log requires strictly positive values, `log1p` buys you zeros, and Yeo-Johnson handles negatives too, which is why it's the safest default. Wait for Chapter 12 for a more thorough answer to "my distribution is too skewed to model."

**Keep, and fix the objective instead.** If the extremes are real and the model is being destabilized by them, the problem is in your loss function—the formula the training process minimizes, which decides what "wrong" costs—not your data. That's what was actually happening to the insurer: MSE was being dominated by large residuals, and the correct response was to change the loss, not to delete the observations generating it. Huber loss behaves like squared error near zero and absolute error in the tails: `HuberRegressor()` for a linear model, `GradientBoostingRegressor(loss="huber")` for trees. Quantile loss lets you model the tail explicitly: `QuantileRegressor(quantile=0.9)`, or `HistGradientBoostingRegressor(loss="quantile", quantile=0.9)`. Modeling `log(y)` and correcting the retransformation bias on the way back (undoing a log with `exp` gives you the median, not the mean; Chapter 12 has the fix) works well for multiplicative processes, and `TransformedTargetRegressor(regressor=model, func=np.log1p, inverse_func=np.expm1)` wires the transform and its inverse into one estimator. Or split the problem: a classifier for "will this be a large loss" plus a separate severity model fit only on the tail—which is, incidentally, a two-part version of the frequency–severity split (model how often claims happen separately from how big each one is) with a large-loss layer that the insurer's own pricing team had been using for thirty years. Nobody asked them.

**Keep, and flag.** Add a binary `is_extreme` column—`df["is_extreme"] = (modified_z.abs() > 3.5).astype(int)`—and let the model decide what to do with it. Same trick as Chapter 10's missingness indicator, same reason it works: you've converted a preprocessing judgment into a feature, which moves the decision from your notebook into the model's fit, where it can be validated.

One thing that should change your defaults: **outlier sensitivity depends on the model.** Tree ensembles (many decision trees voting or averaging, gradient-boosted trees included) barely notice extreme feature values, because a split at "x > 400" behaves identically whether the point is 401 or 4,000,001—trees use order, not magnitude. Linear models, SVMs, k-NN (which scores a point by its nearest neighbors), PCA (Chapter 15's dimension reduction), k-means (clustering around averages), and neural networks are all acutely sensitive. If one of those is your model, `RobustScaler()`—center on the median, divide by the IQR—puts features on a common scale without letting a few extremes set it. So the real question is not "should I handle outliers" but "does my model care?" A great deal of outlier treatment gets applied reflexively ahead of a gradient-boosted tree, where it accomplishes nothing except throwing away data.

---

## 11.5 Industry-Specific Outlier Handling

The right answer varies more by domain than by technique, and the pattern across domains is uncomfortable: in most of the fields where this matters, the outlier is the point.

**Fraud and security.** The outlier *is* the target. Any preprocessing step that removes outliers is removing your positive class (the rows carrying the label you're trying to catch). Teams lose months wondering why their fraud model finds no fraud, having diligently cleaned the training set first. Compounding it: adversaries adapt toward normal. Yesterday's obvious anomaly is today's carefully-shaped ordinary transaction, which is Chapter 9's concept drift—the relationship between inputs and target shifting under you—with a motive.

**Manufacturing and predictive maintenance.** The excursions are precursors. A vibration spike four hours before a bearing seizes is not noise contaminating your dataset; it is the dataset. Remove pre-failure anomalies and you have built a model that describes machines that are working fine, which you did not need.

**Finance and insurance.** Fat tails are structural, not anomalous. The entire discipline of risk management exists because the tail is where the money is, and a model assuming Gaussian (bell-curve) returns is wrong in a way that only becomes relevant on the days when being wrong is expensive. The cold open is the general case, not a special one.

**Healthcare.** A blood pressure of 40 over 20 is either a patient in shock or a cuff that fell off the arm. Identical number, opposite responses, and the only thing that separates them is context from elsewhere in the record. This is the domain where automated outlier removal is most dangerous, because the anomalous vital sign is the reason the system exists.

**Retail and e-commerce.** Black Friday is not an outlier. It's a Tuesday in November, it recurs annually, and any detector without a seasonal baseline will flag your best day of the year as corrupt data. The genuine outliers here are usually different populations rather than errors: wholesale buyers in a consumer dataset, employees using a staff discount, resellers.

**Advertising and web analytics.** Bots. Enormous volume, entirely non-human, and not errors—they are correct records of a thing you don't want to model. Segment them out with a mechanism (user agent, behavioral fingerprint), not with a threshold on session count.

**Scientific and experimental data.** The outlier may be the discovery. There's a long, embarrassing history of results getting cleaned out of datasets because they didn't fit the expected distribution. If you're in research, the burden of proof for deletion should be higher than anywhere else on this list.

---

## 11.6 Real-time Outlier Detection Systems

Batch detection has the luxury of seeing the whole distribution. Streaming detection has to decide about a point the instant it arrives, using a baseline that is itself moving, and that changes almost every design decision.

**The baseline is the hard part.** You need a running notion of normal, and it has to adapt. Rolling windows (`s.rolling("7D").mean()`) and exponentially weighted moving averages (`s.ewm(halflife="1D", times=s.index).mean()`, an average that weights recent points more and lets old ones fade) are the standard tools, and the window length is a genuine trade-off, not a tuning detail. Too short and a slow anomaly gets absorbed into the baseline—the detector learns the problem as normal and goes quiet. Too long and it takes weeks to adapt to a legitimate shift, so it screams through every product launch.

**Seasonality will eat you alive.** Compare traffic to the last hour and you flag every Monday at 9 a.m., every batch job at midnight, every payday. The baseline needs to be same-hour-last-week (`s - s.shift(freq="7D")`), or explicitly decomposed into trend plus seasonal plus residual (`statsmodels.tsa.seasonal.STL`) with detection running on the residual. Most streaming detectors that get switched off in their first month get switched off for this reason.

**Cold start.** A new customer, a new device, a new merchant has no history, and that's precisely when you most want to know if something's wrong. Fall back to a cohort baseline—what's normal for accounts in their first week—and transition to the entity's own baseline as history accumulates. Decide explicitly what happens in the gap, because otherwise the answer is "everything looks normal," which is the wrong default for an unknown entity.

**Watch out for the poisoning loop.** If the baseline updates from a stream that includes the anomalies, a sufficiently patient attacker or a slow-drifting fault will teach the detector to accept it. Ten percent worse every week is invisible forever. Update baselines from data that has already been screened, or hold suspected anomalies out of the baseline update until a human confirms them.

**Alert fatigue is a data-quality failure, not a people problem.** A detector with a 5% false-positive rate (the share of normal events it wrongly flags) on ten million events a day produces five hundred thousand alerts. Nobody reads five hundred thousand of anything. The design target is not detection rate; it's the number of alerts a human can actually act on in a day, which is usually somewhere between ten and fifty. Pick that number first, then set your threshold to produce it, then measure what you miss. A detector nobody reads has a true detection rate of zero regardless of what the offline evaluation said.

**Quarantine, don't drop.** The best production pattern available: when a record looks anomalous, route it to a side table along with the detector score and the reason, let the main pipeline continue, and review the quarantine. You keep the evidence, you don't block the pipeline, and—the part that pays off later—you are accumulating a labeled dataset. Every quarantined record a human reviews becomes a label, and once you have enough labels you can replace the unsupervised detector with a supervised model that's dramatically better. Dropping the row throws away the raw material for the thing you actually want.

---

## Quick Wins: Outlier Checks Before You Delete Anything

**Count what your cleaning code removes (10 min).** Search your preprocessing for `clip`, `winsorize`, `quantile`, `drop`, and `3 *`. For every hit, print how many rows or values it affects and what fraction of the total that is. Dollars to donuts, you're going to be surprised. If any single step touches more than 1% of your data, it deserves a written justification.

**Plot on a log scale (5 min).** Take whatever variable is generating your outlier complaints and histogram `log(x)` instead of `x`: `np.log1p(x).plot.hist(bins=50)`. A large share of the time the terrifying skew resolves into a clean, nearly symmetric distribution with no outliers at all, and your problem was never outliers—it was that you were looking at a multiplicative process on a linear axis.

**Sum the value in your tail (15 min).** Sort by whatever the business cares about—revenue, loss, cost, duration—and compute what fraction of the total sits in the top 1%: `x.nlargest(len(x) // 100).sum() / x.sum()`. If that number is large, and in most businesses it is, write it on the wall next to whoever owns the preprocessing. It's the number that tells you whether outlier removal is cleaning or amputation.

**Check whether your model even cares (20 min).** Train once with your outlier treatment and once without, on the same split. If the metrics are identical—which they frequently are with tree ensembles—delete the preprocessing step, not the data. You've been paying a complexity and correctness cost for nothing.

---

## Your Homework

### Exercise 1: The Deletion Audit (Time: ~45 minutes)

Take your production training pipeline and instrument every step that removes or modifies a value on the grounds that it's extreme. For each one, write down three things: how many records it affects, what mechanism justifies it, and who decided. Most pipelines have three to six of these, most of them were added during a deadline by someone who has since left, and most of them have no recorded justification at all. The ones you can't explain are the ones to remove first—not because they're necessarily wrong, but because an unexplained rule is one nobody can evaluate.

### Exercise 2: Find a Multivariate Outlier (Time: ~1 hour)

Pick two or three related columns in your data—ones that should move together. Screen each individually for outliers with a modified z-score and note what you catch. Then fit a robust Mahalanobis distance or an Isolation Forest across the combination and note what *that* catches. Pull the top twenty records that the multivariate method flagged and the univariate methods missed, and go look at them by hand. In every dataset a professional has ever run this on, at least one of those twenty turned out to be a real bug in an upstream system.

### Exercise 3: Rebuild the Insurer's Mistake (Time: ~45 minutes, instructive)

Take any dataset with a skewed target. Train three models: one on the raw target, one with the target winsorized at the 99th percentile, and one on `log(target)` with the predictions transformed back. Evaluate all three on the *uncapped* test set, and then evaluate them again on only the top 1% of the test set by target value. The capped model will look competitive overall and catastrophic on the tail. That gap is exactly the $26 million, and the reason it stays hidden in real projects is that nobody reports metrics segmented by target magnitude. Start doing that.

---

## Bridge to Chapter 12

Half the outliers in this chapter weren't outliers. They were the correct upper tail of a skewed distribution, viewed on a linear axis by a model that assumes symmetry, and the appropriate response was never deletion—it was to change the scale.

That's Chapter 12. Feature scaling and transformation is boring but it silently decides whether half the algorithms in your toolkit work at all. A k-nearest-neighbors model where one feature is measured in dollars and another in years isn't computing similarity, it's computing dollars. Gradient descent (the step-by-step weight-nudging that trains linear models and neural networks) on unscaled features doesn't converge so much as stagger. And the log transform that makes your claim-severity distribution behave is the same operation that makes your outlier problem disappear without deleting a single row.

Chapter 12 covers which algorithms actually require scaling and which don't care, how to pick a transformation, and the leakage trap that sits inside every scaler—the one where you fit it on everything you have and inflate every number you report afterward.

---

*P.S. — The most dangerous thing about the three-sigma rule is that it works often enough to become a habit. You apply it, the histogram tidies up, the model trains faster, the metric improves, and nothing bad happens, so you apply it again next project. It fails silently and only in the tail, which is the one region nobody is monitoring, and by the time it costs you real money the preprocessing step is four years old and nobody remembers writing it. Somewhere in your codebase is a `clip()` call that a contractor added in 2021 to make a chart look nicer. Go find it.*
