# Chapter 11: Outlier Detection and Treatment

## Or: The Number You Deleted Was the Business

A property and casualty insurer I advised wrote about 1.2 million claims a year across personal auto and homeowners, against roughly $620 million in annual incurred losses. They built a claim-severity model: at first notice of loss, predict what this claim will ultimately cost, so the reserve gets set correctly on day one instead of six months later when an adjuster finally gets to it. Good project. Real money attached to it.

The model validated at an R² of 0.71, which for claim severity is a strong number. It went into production setting initial reserves.

Eleven months later the year-end actuarial review came back with a reserve strengthening charge of $26 million. The book had been systematically under-reserved all year, and the model had been confidently wrong in one direction the entire time.

The cause was three lines in a preprocessing notebook:

```python
cap = df["ultimate_loss"].quantile(0.99)
df["ultimate_loss"] = df["ultimate_loss"].clip(upper=cap)
```

Someone had winsorized the target at the 99th percentile. And I want to be careful here, because the person who wrote that was not being lazy. They were solving a real problem, correctly diagnosed. Claim severity is violently heavy-tailed, mean squared error is dominated by the largest residuals, and a handful of catastrophic claims were jerking the training around so badly the model wouldn't converge cleanly. Capping the target fixed it. The loss curve got smooth. The validation metric went up. Every signal a data scientist normally trusts said this was the right move.

In personal lines, the top 1% of claims by size accounted for about 34% of total loss dollars. A total house fire, a severe bodily injury with a lifetime care component, a liability verdict—these are rare and they are most of the money. Capping at the 99th percentile removed every example of the phenomenon that determines whether the company is profitable. The model was trained on a world in which the largest possible claim was $180,000, and then deployed into a world that regularly produces claims of $2 million.

So it did exactly what it was taught. It became superb at predicting fender-benders and water damage—the claims that are numerous, well-behaved, and financially irrelevant—and it had never once seen a severe claim, so it reserved every one of them at roughly the cap. Twelve thousand times a year it looked at a catastrophic loss and said "call it a hundred and eighty grand." The R² of 0.71 was real. It was measured on capped data, against a capped target, which is to say it was a precise measurement of the model's skill at a task nobody needed done.

They deleted the top 1% of their data and with it about a third of their business.

---

The reflex that produced that disaster is the same reflex that produced Chapter 10's. Confronted with a value that makes the data awkward, reach for the standard move that makes it go away. Fill the null. Clip the tail. In both cases the standard move is a bet about where the value came from, and in both cases the pipeline will run perfectly whether or not the bet was right.

An outlier is not a property of a number. It is a *claim about the process that generated the number*—specifically, the claim that this point came from somewhere your model shouldn't represent. That claim can be true. Frequently it is. But it's a claim about the world, it cannot be settled by a threshold, and the entire cost of getting it wrong lands in the tail, which is where a great many businesses keep their profit, their risk, and the thing they hired you to predict.

---

## 11.1 Defining Outliers: Statistical vs Domain-Based Approaches

The three-sigma rule is the most widely used piece of statistics in industry and almost nobody remembers where it comes from. Under a normal distribution, 99.7% of mass sits within three standard deviations of the mean, so a point outside that range is rare—one in about 370. Flagging it as suspicious is reasonable.

*Under a normal distribution.* That's the whole load-bearing clause, and it does not hold for most of the quantities anyone cares about. Claim sizes, incomes, transaction values, page views, session durations, file sizes, city populations, word frequencies, order quantities—all right-skewed, most of them roughly log-normal or worse. On a log-normal distribution, points five and six standard deviations above the mean are not anomalies. They are Tuesday. Apply a three-sigma filter to a heavy-tailed variable and you are not removing errors, you are performing a distributional lobotomy and calling it data cleaning.

The mean and standard deviation have a second problem, which is that they are computed from the data including the outliers. One value entered as 5,000,000 instead of 50 will inflate the standard deviation enough that it no longer flags itself. Statisticians call this *masking*, and it means the three-sigma rule is least reliable in exactly the situation you deployed it for.

The **IQR or Tukey fence** (flag anything below Q1 − 1.5×IQR or above Q3 + 1.5×IQR) is better on both counts, because quartiles don't care how extreme the extreme values are. It's what the whiskers on a box plot mean. But it was designed with roughly symmetric distributions in mind, and on a strongly right-skewed variable it will flag a large slice of the upper tail *by construction*—not because anything is wrong, but because the fence assumes a symmetry the data doesn't have. Run it on income data and watch it declare that several percent of the population is anomalous.

The **modified z-score** built on median absolute deviation is the best of the simple options. Take the median, take the median of the absolute deviations from that median, and scale: 0.6745 × (x − median) / MAD, flagging above about 3.5. MAD has a breakdown point of 50%, meaning half your data would have to be corrupted before it lies to you. Compare that to the standard deviation, where one bad value is enough. If you're going to use a threshold, use this one.

But all of this is procedure, and procedure is the smaller half of the job. Every extreme value in your dataset is one of three things, and the statistics cannot tell you which:

**It's an error.** A decimal point in the wrong place, a unit mix-up, a sensor reporting garbage, an undecoded sentinel like -999 or 9999 masquerading as a measurement (Chapter 2's problem, arriving late and expensive). Remove it or repair it.

**It's a different population.** A wholesale account sitting in a retail dataset. A bot in a table of human users. An internal test transaction that was never filtered out. This point is not wrong—it is correct data about a thing you weren't modeling. The right move is almost never deletion; it's segmentation, or a flag, or a separate model.

**It's a rare real event.** The house fire. The machine three hours from failure. The fraudulent transaction. The point is correct, it belongs to the population you're modeling, and it is very often the point you are being paid to predict.

Which gives you the only rule in this chapter I'd defend without qualification: **you may delete a data point only if you can name the mechanism that made it wrong.** "It's 4.2 sigma from the mean" is not a mechanism. "The vendor changed the unit from liters to gallons in March and this batch wasn't converted" is a mechanism. If you can't name one, you are not removing an error—you are removing evidence, and you should keep the row and go find out what it is.

---

## 11.2 Univariate and Multivariate Detection Methods

Everything in 11.1 is univariate: one column at a time. That's where nearly all production outlier handling lives, and it has a blind spot large enough to drive the whole problem through.

Consider a dataset of people. A height of 6'2" is unremarkable. A weight of 110 pounds is unremarkable. A 6'2" person weighing 110 pounds is medically extraordinary and almost certainly a data-entry error—and **no column-by-column check will ever find it**, because neither value is unusual on its own. The anomaly lives in the relationship, not in either number.

Your data is full of these. A customer with a two-week account age and a $40,000 lifetime spend. A ninety-minute session with one page view. A claim with a severe injury code and a $400 payout. Every one of them is invisible to a per-column scan, and every one of them is more interesting than anything a per-column scan will surface.

**Mahalanobis distance** is the classical answer: measure distance from the multivariate center in units of the covariance structure, so the metric knows that height and weight travel together. Two caveats, both serious. It assumes roughly elliptical structure, so it struggles with clusters and nonlinear relationships. And it's computed from the sample mean and covariance matrix, which the outliers themselves corrupt—masking again, worse in higher dimensions. If you use it, use a robust estimator underneath, such as Minimum Covariance Determinant, which fits the center and covariance on the tightest subset of the data and then measures everything against that.

Then there's the ceiling nobody warns you about. In high dimensions, distance stops discriminating. As you add dimensions, the distances between all pairs of points converge toward each other, and "far from the center" degrades into noise. Distance-based outlier detection works well in five dimensions, works poorly in fifty, and is essentially decorative in five hundred. If you're operating in high dimensions, reduce first (Chapter 15) or use a method that doesn't depend on a global distance metric.

Two more categories worth having names for, because they change what you look for:

**Contextual outliers** are only anomalous given their context. Forty degrees Fahrenheit is normal in January and alarming in July. A $5,000 charge is routine for one cardholder and a red flag for another. No global threshold captures this—you have to condition on time, entity, or segment, which means the detector needs the same point-in-time discipline as everything else in this book.

**Collective outliers** are anomalous as a group while every individual point looks fine. The classic is a sensor stream that goes perfectly flat: every reading is inside the normal range, and the *absence of variance* is the failure. A vibration sensor reporting exactly 0.42 for six straight hours has died, and every univariate check in the world will wave it through. Watch second-order properties—variance, autocorrelation, the rate of change—not just values.

---

## 11.3 Machine Learning-Based Anomaly Detection

When thresholds and distances run out, there's a family of methods that learn the shape of "normal" and score deviations from it.

**Isolation Forest** is the one I reach for first, because its core idea is clever and it scales. Build random trees by picking a random feature and a random split point, over and over. Anomalies get isolated in very few splits—there's not much data around them, so a random cut separates them quickly—while normal points require many splits to fence off. The score is average path length. It's roughly linear in the number of samples, it needs no distance metric, and it handles moderate dimensionality far better than anything distance-based. Its `contamination` parameter is a trap: it asks you to declare what fraction of your data is anomalous, which is the thing you were trying to find out. Set it deliberately or leave it and threshold the raw scores yourself.

**Local Outlier Factor** compares a point's local density to the density of its neighbors. This catches something global methods structurally cannot: a point sitting in a sparse pocket that isn't far from the overall center. If your data has clusters of different densities—and real data does—LOF finds anomalies that Mahalanobis and Isolation Forest both miss.

**One-Class SVM** learns a boundary enclosing the normal region. Powerful and principled, sensitive to kernel and `nu` choices, and it scales poorly. Useful when you have a clean sample of definitely-normal data to fit on.

**Autoencoder reconstruction error** is the deep-learning entry: train a network to compress and reconstruct normal data, then flag whatever it reconstructs badly. This works well on images, high-dimensional sensor arrays, and sequences, where the notion of "distance" is useless but the notion of "this doesn't look like the things I was trained on" is exactly right. It carries all the operational baggage from Chapter 10—it's a second model, it drifts, it needs monitoring.

**DBSCAN** gives you outliers as a side effect of clustering: points that never join a cluster are noise, which is often what you wanted.

Now the structural limitation that applies to all of them. These are unsupervised methods, which means they answer the question "is this statistically unusual?" That is not the question. The question is "is this *wrong*, or is it the rare real thing I care about?" Nothing unsupervised can distinguish those, because the difference isn't in the data, it's in the mechanism. An anomaly detector produces a ranked list of things to look at. It does not produce a decision, and the moment you wire its output directly into `df.drop()` you've automated a judgment nobody made.

And the corollary that people miss constantly: **if you have labels, use them.** If you know which transactions were fraud, which machines failed, which claims blew up—train a supervised classifier. It will beat every anomaly detector on this list, and it isn't close. Anomaly detection is the technique you use when you don't have labels. Reaching for Isolation Forest on a labeled problem because it sounds more like anomaly detection is a mistake I see roughly monthly.

---

## 11.4 Treatment Strategies: Remove, Cap, Transform, or Keep

You found them. Now what? There are five real options and the wrong one costs $26 million.

**Remove.** Justified only when you can name the mechanism—the rule from 11.1. Three disciplines make removal safe. Log every deletion with its reason, to a file, not to a comment. Count them as a percentage of your data; if you're dropping 5%, you have stopped cleaning and started reshaping. And remember that deleting a row in training does nothing to the world: whatever produced that value will produce it again next Tuesday, in production, to a model that has now never seen one.

**Cap or winsorize.** Bound the value without losing the row. Legitimate when the quantity is genuinely bounded and the excess is definitionally impossible—an age of 214, a body temperature of 900. Illegitimate when the tail carries the meaning. The question that decides it is one most people never ask: **are you capping a feature or the target?** Capping a feature limits how much leverage one observation has over the fit, which is often exactly right. Capping the target changes what the model is being asked to predict, and if the tail is where your economics live, you have just redefined the problem into one that doesn't matter. The insurer capped the target.

**Transform.** Apply a log, a Box-Cox, a Yeo-Johnson, or a quantile transform. This is usually the right answer and it's underused because it feels less decisive than deletion. A log transform compresses a heavy tail into something a linear model can work with while preserving every row and every ordering—the largest claim is still the largest claim, it just no longer dominates the squared error. Log requires strictly positive values; Yeo-Johnson handles zeros and negatives, which is why it's the safer default. This is Chapter 12's entire subject, and it's the most common correct answer to "my distribution is too skewed to model."

**Keep, and fix the objective instead.** If the extremes are real and the model is being destabilized by them, the problem is in your loss function, not your data. That's what was actually happening to the insurer: MSE was being dominated by large residuals, and the correct response was to change the loss, not to delete the observations generating it. Huber loss behaves like squared error near zero and absolute error in the tails. Quantile loss lets you model the tail explicitly. Modeling `log(y)` and correcting the retransformation bias on the way back works well for multiplicative processes. Or split the problem: a classifier for "will this be a large loss" plus a separate severity model fit only on the tail—which is, incidentally, standard actuarial practice that the insurer's own pricing team had been using for thirty years. Nobody asked them.

**Keep, and flag.** Add a binary `is_extreme` column and let the model decide what to do with it. Same trick as Chapter 10's missingness indicator, same reason it works: you've converted a preprocessing judgment into a feature, which moves the decision from your notebook into the model's fit, where it can be validated.

One thing that should change your defaults: **outlier sensitivity depends on the model.** Tree ensembles barely notice extreme feature values, because a split at "x > 400" behaves identically whether the point is 401 or 4,000,001—trees use order, not magnitude. Linear models, SVMs, k-NN, PCA, k-means, and neural networks are all acutely sensitive. So the real question is not "should I handle outliers" but "does my model care?" A great deal of outlier treatment gets applied reflexively ahead of a gradient-boosted tree, where it accomplishes nothing except throwing away data.

---

## 11.5 Industry-Specific Outlier Handling

The right answer varies more by domain than by technique, and the pattern across domains is uncomfortable: in most of the fields where this matters, the outlier is the point.

**Fraud and security.** The outlier *is* the target. Any preprocessing step that removes outliers is removing your positive class, and I have watched a team spend a month wondering why their fraud model found no fraud, having cleaned the training set first. Compounding it: adversaries adapt toward normal. Yesterday's obvious anomaly is today's carefully-shaped ordinary transaction, which is Chapter 9's concept drift with a motive.

**Manufacturing and predictive maintenance.** The excursions are precursors. A vibration spike four hours before a bearing seizes is not noise contaminating your dataset; it is the dataset. Remove pre-failure anomalies and you have built a model that describes machines that are working fine, which you did not need.

**Finance and insurance.** Fat tails are structural, not anomalous. The entire discipline of risk management exists because the tail is where the money is, and a model assuming Gaussian returns is wrong in a way that only becomes relevant on the days when being wrong is expensive. The cold open is the general case, not a special one.

**Healthcare.** A blood pressure of 40 over 20 is either a patient in shock or a cuff that fell off the arm. Identical number, opposite responses, and the only thing that separates them is context from elsewhere in the record. This is the domain where automated outlier removal is most dangerous, because the anomalous vital sign is the reason the system exists.

**Retail and e-commerce.** Black Friday is not an outlier. It's a Tuesday in November, it recurs annually, and any detector without a seasonal baseline will flag your best day of the year as corrupt data. The genuine outliers here are usually different populations rather than errors: wholesale buyers in a consumer dataset, employees using a staff discount, resellers.

**Advertising and web analytics.** Bots. Enormous volume, entirely non-human, and not errors—they are correct records of a thing you don't want to model. Segment them out with a mechanism (user agent, behavioral fingerprint), not with a threshold on session count.

**Scientific and experimental data.** The outlier may be the discovery. There's a long, embarrassing history of results getting cleaned out of datasets because they didn't fit the expected distribution. If you're in research, the burden of proof for deletion should be higher than anywhere else on this list.

---

## 11.6 Real-time Outlier Detection Systems

Batch detection has the luxury of seeing the whole distribution. Streaming detection has to decide about a point the instant it arrives, using a baseline that is itself moving, and that changes almost every design decision.

**The baseline is the hard part.** You need a running notion of normal, and it has to adapt. Rolling windows and exponentially weighted moving averages are the standard tools, and the window length is a genuine trade-off, not a tuning detail. Too short and a slow anomaly gets absorbed into the baseline—the detector learns the problem as normal and goes quiet. Too long and it takes weeks to adapt to a legitimate shift, so it screams through every product launch.

**Seasonality will eat you alive.** Compare traffic to the last hour and you flag every Monday at 9 a.m., every batch job at midnight, every payday. The baseline needs to be same-hour-last-week, or explicitly decomposed into trend plus seasonal plus residual with detection running on the residual. Most streaming detectors that get switched off in their first month get switched off for this reason.

**Cold start.** A new customer, a new device, a new merchant has no history, and that's precisely when you most want to know if something's wrong. Fall back to a cohort baseline—what's normal for accounts in their first week—and transition to the entity's own baseline as history accumulates. Decide explicitly what happens in the gap, because otherwise the answer is "everything looks normal," which is the wrong default for an unknown entity.

**Watch out for the poisoning loop.** If the baseline updates from a stream that includes the anomalies, a sufficiently patient attacker or a slow-drifting fault will teach the detector to accept it. Ten percent worse every week is invisible forever. Update baselines from data that has already been screened, or hold suspected anomalies out of the baseline update until a human confirms them.

**Alert fatigue is a data-quality failure, not a people problem.** A detector with a 5% false-positive rate on ten million events a day produces five hundred thousand alerts. Nobody reads five hundred thousand of anything. The design target is not detection rate; it's the number of alerts a human can actually act on in a day, which is usually somewhere between ten and fifty. Pick that number first, then set your threshold to produce it, then measure what you miss. A detector nobody reads has a true detection rate of zero regardless of what the offline evaluation said.

**Quarantine, don't drop.** The best production pattern I know: when a record looks anomalous, route it to a side table along with the detector score and the reason, let the main pipeline continue, and review the quarantine. You keep the evidence, you don't block the pipeline, and—the part that pays off later—you are accumulating a labeled dataset. Every quarantined record a human reviews becomes a label, and once you have enough labels you can replace the unsupervised detector with a supervised model that's dramatically better. Dropping the row throws away the raw material for the thing you actually want.

---

## Quick Wins: Outlier Checks Before You Delete Anything

**Count what your cleaning code removes (10 min).** Search your preprocessing for `clip`, `winsorize`, `quantile`, `drop`, and `3 *`. For every hit, print how many rows or values it affects and what fraction of the total that is. People are routinely startled. If any single step touches more than 1% of your data, it deserves a written justification.

**Plot on a log scale (5 min).** Take whatever variable is generating your outlier complaints and histogram `log(x)` instead of `x`. A large share of the time the terrifying skew resolves into a clean, nearly symmetric distribution with no outliers at all, and your problem was never outliers—it was that you were looking at a multiplicative process on a linear axis.

**Sum the value in your tail (15 min).** Sort by whatever the business cares about—revenue, loss, cost, duration—and compute what fraction of the total sits in the top 1%. If that number is large, and in most businesses it is, write it on the wall next to whoever owns the preprocessing. It's the number that tells you whether outlier removal is cleaning or amputation.

**Check whether your model even cares (20 min).** Train once with your outlier treatment and once without, on the same split. If the metrics are identical—which they frequently are with tree ensembles—delete the preprocessing step, not the data. You've been paying a complexity and correctness cost for nothing.

---

## Your Homework

### Exercise 1: The Deletion Audit (Time: ~45 minutes)

Take your production training pipeline and instrument every step that removes or modifies a value on the grounds that it's extreme. For each one, write down three things: how many records it affects, what mechanism justifies it, and who decided. Most pipelines have three to six of these, most of them were added during a deadline by someone who has since left, and most of them have no recorded justification at all. The ones you can't explain are the ones to remove first—not because they're necessarily wrong, but because an unexplained rule is one nobody can evaluate.

### Exercise 2: Find a Multivariate Outlier (Time: ~1 hour)

Pick two or three related columns in your data—ones that should move together. Screen each individually for outliers with a modified z-score and note what you catch. Then fit a robust Mahalanobis distance or an Isolation Forest across the combination and note what *that* catches. Pull the top twenty records that the multivariate method flagged and the univariate methods missed, and go look at them by hand. In every dataset I have ever run this on, at least one of those twenty turned out to be a real bug in an upstream system.

### Exercise 3: Rebuild the Insurer's Mistake (Time: ~45 minutes, instructive)

Take any dataset with a skewed target. Train three models: one on the raw target, one with the target winsorized at the 99th percentile, and one on `log(target)` with the predictions transformed back. Evaluate all three on the *uncapped* test set, and then evaluate them again on only the top 1% of the test set by target value. The capped model will look competitive overall and catastrophic on the tail. That gap is exactly the $26 million, and the reason it stays hidden in real projects is that nobody reports metrics segmented by target magnitude. Start doing that.

---

## Bridge to Chapter 12

Half the outliers in this chapter weren't outliers. They were the correct upper tail of a skewed distribution, viewed on a linear axis by a model that assumes symmetry, and the appropriate response was never deletion—it was to change the scale.

That's Chapter 12. Feature scaling and transformation is the most boring topic in this book and it silently decides whether half the algorithms in your toolkit work at all. A k-nearest-neighbors model where one feature is measured in dollars and another in years isn't computing similarity, it's computing dollars. Gradient descent on unscaled features doesn't converge so much as stagger. And the log transform that makes your claim-severity distribution behave is the same operation that makes your outlier problem disappear without deleting a single row.

Chapter 12 covers which algorithms actually require scaling and which don't care, how to pick a transformation, and the leakage trap that sits inside every scaler—the one where you fit it on everything you have and inflate every number you report afterward.

---

*P.S. — The most dangerous thing about the three-sigma rule is that it works often enough to become a habit. You apply it, the histogram tidies up, the model trains faster, the metric improves, and nothing bad happens, so you apply it again next project. It fails silently and only in the tail, which is the one region nobody is monitoring, and by the time it costs you real money the preprocessing step is four years old and nobody remembers writing it. Somewhere in your codebase is a `clip()` call that a contractor added in 2021 to make a chart look nicer. Go find it.*
