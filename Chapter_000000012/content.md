# Chapter 12: Data Transformation and Scaling

## Or: Your Model Was Measuring Pounds

A freight marketplace I advised moved about 900 shipments a day. Shippers posted loads, carriers bid, and the platform's job was to recommend which carriers to surface for a given load. Get it right and the freight arrives on time and undamaged; get it wrong and you've handed a fragile pallet to the cheapest guy with a truck.

They built a matching model on five features: distance in miles, weight in pounds, the carrier's on-time percentage, years in business, and a safety rating from 1 to 5 that the operations team had spent a year and a half building. The safety rating was the whole point. It was the thing that made the marketplace worth more than a spreadsheet of phone numbers.

The model used k-nearest-neighbors to find comparable historical matches. It shipped. And for fourteen months it was, by the standard of the metric anyone was watching, fine.

Then the claims data got reviewed properly. Damage and late-delivery claims had reached $4.6 million a year, concentrated in a set of carriers that the safety rating had flagged as poor from the beginning. The model had been recommending them the entire time.

Here's the arithmetic that explains it. Euclidean distance sums the squared differences across features. Weight ranged from about 200 pounds to 45,000. Safety rating ranged from 1 to 5. So when the model compared two candidate matches, a difference of **four pounds** of freight contributed as much to the distance calculation as the entire gap between the safest carrier in the network and the most dangerous one.

The safety rating wasn't underweighted. It was *arithmetically invisible*. Eighteen months of operations work, reduced to rounding error by a unit of measurement. The model was not matching on quality, or reliability, or tenure. It was matching on weight, with a rounding correction for distance, and everything else in that feature vector was decorative.

The fix was one line:

```python
X = StandardScaler().fit_transform(X)
```

On-time delivery went from 82% to 91% in the following quarter. Nobody retrained anything, nobody added a feature, nobody touched the architecture. They just stopped measuring quality in pounds.

---

I've put this chapter's disaster up front the same way as the others, but I want to name what makes this one different. Chapters 10 and 11 were about judgment—someone made a defensible call about nulls or outliers and the call was wrong. This one isn't judgment. Nobody decided the safety rating should be ignored. The decision was made by the numeric range of a column, on their behalf, silently, and it held for fourteen months because there is no error message for "your distance metric is in the wrong units."

That's what makes scaling the most boring dangerous topic in this book. It fails without symptoms. The model trains, the metrics compute, the dashboard is green, and the only evidence is a business outcome that nobody thinks to connect back to a preprocessing step.

---

## 12.1 Feature Scaling: Algorithm Requirements and Performance Impact

The first thing to get straight is that "should I scale?" has an actual answer, and it depends entirely on how your algorithm consumes the numbers. There are three groups.

**Algorithms that break without scaling.** Anything that computes a distance: k-nearest-neighbors, k-means, hierarchical clustering, DBSCAN, and SVMs with an RBF kernel. All of them share the cold open's failure mode—the feature with the widest numeric range becomes the metric, and everything else becomes noise. Also in this group: anything trained by gradient descent, including every neural network. When features have wildly different scales, the loss surface becomes a long narrow valley, and gradient descent bounces off the walls instead of running down the floor. You'll see it as slow convergence, or as a learning rate that's simultaneously too large for one feature and too small for another, which no amount of tuning fixes because it's not a learning-rate problem.

**Algorithms where scaling changes the answer in a way people miss.** This is the group that costs the most, because the model still *works*—it just works on a different problem than you think.

Regularized regression is the big one. Ridge, Lasso, and ElasticNet all penalize the magnitude of coefficients. But a coefficient's magnitude depends on its feature's units. Measure a distance in miles and its coefficient is some number; measure the same distance in feet and the coefficient is 5,280 times smaller, so the penalty barely touches it. **You are not regularizing your features, you are regularizing your unit choices.** Run Lasso on unscaled data and the variables it zeroes out are substantially determined by whoever picked the units upstream, which is usually a database schema written by someone who has never heard of your model. This is the most common serious scaling bug I find in production code, and unlike the freight disaster it produces a model that looks completely reasonable.

PCA has the same disease for the same reason. PCA finds directions of maximum variance, and variance is in squared units, so the component structure is dominated by whichever feature has the largest numeric spread. Unscaled PCA on a table with a revenue column returns, essentially, the revenue column with extra steps. (Chapter 15 goes further into this.)

**Algorithms that don't care at all.** Decision trees and every ensemble built on them—random forests, XGBoost, LightGBM, CatBoost. A tree splits on *order*, not magnitude. The rule "weight > 12,000" partitions the data identically whether weight is in pounds, kilograms, or tons; the threshold just changes to match. Any monotonic transformation of a feature produces the same tree. This is why gradient-boosted trees are so forgiving of raw messy tabular data, and it's why a large amount of the scaling code in the world is doing nothing at all.

That last point deserves emphasis, because the reflex runs both directions. Roughly as often as I find a k-NN model measuring pounds, I find a `StandardScaler` sitting in front of an XGBoost model, adding a fitted artifact that has to be versioned, shipped, and kept in sync between training and serving—for zero benefit. It's not harmful to the math. It's harmful to the system, because every fitted preprocessing step is another thing that can drift out of sync between your training and serving paths, which is Chapter 9's entire disaster waiting for a chance.

So the question isn't "should I scale." It's "does my algorithm use magnitude, distance, or a penalty term?" If yes, scaling is mandatory. If no, scaling is a liability you're carrying for aesthetic reasons.

---

## 12.2 Core Scaling Techniques and When to Use Them

Six things people call scalers. They are not interchangeable, and two of them are routinely used by accident.

**StandardScaler**—subtract the mean, divide by the standard deviation. Output has mean 0 and standard deviation 1, unbounded in both directions. The default, correctly. Use it when your feature is roughly symmetric and you don't have wild outliers. Note what it does *not* do: it does not make anything normally distributed. A bimodal feature standardized is still bimodal, now centered at zero. People say "normalize" and mean this, and then assume normality they didn't get.

**MinMaxScaler**—subtract the min, divide by the range, output bounded to [0, 1]. Useful when you need a hard bound: image pixel values, neural network inputs where a bounded activation expects it. And it is catastrophically fragile, because the min and max are the two most outlier-sensitive statistics in existence. One data-entry error of 9,999,999 in your training set and every real value gets squashed into the bottom 0.001% of the range. You have not scaled your data; you have converted it into a constant with one interesting point. Chapter 11's material is a prerequisite for using this safely.

**RobustScaler**—subtract the median, divide by the interquartile range. Same idea as StandardScaler, built from statistics that don't care about the tails. When you have outliers you've decided to keep (which, per Chapter 11, is often the right call), this is the scaler that lets you keep them without letting them set the scale for everything else. Underused.

**MaxAbsScaler**—divide by the maximum absolute value, output in [-1, 1]. Its one distinguishing property: it doesn't shift the data, so zeros stay zeros and sparse matrices stay sparse. If you're working with high-dimensional sparse data—text features, one-hot encodings at scale—running StandardScaler will center the data, destroy sparsity, and turn a 2 GB sparse matrix into a 400 GB dense one. This is a real way to run a machine out of memory.

**Normalizer**—and this is the one people use by accident. Every other scaler on this list works down a *column*, one feature at a time, fit across your rows. `Normalizer` works across a *row*, scaling each sample to unit norm. It's a completely different operation for a completely different purpose (comparing direction rather than magnitude, standard in text similarity). It is next to `StandardScaler` in the docs, it sounds like the generic option, and if you drop it into a tabular pipeline it will silently destroy the relationship between your features. Check your imports.

**QuantileTransformer**—map each value to its rank, then to a uniform or normal distribution. This is the heavy artillery: it's immune to outliers by construction, and it will force any input distribution into the output shape you asked for, no matter how ugly the input was. The costs are real. It discards the *shape* of the distribution and keeps only the ordering, so the distance between two points stops meaning what it used to mean. It needs enough data to estimate quantiles reliably. And it will happily map a value it never saw during training onto the edge of the distribution, which makes its production behavior on new extremes worth testing deliberately.

The practical default: **RobustScaler if you have outliers, StandardScaler if you don't, MaxAbsScaler if you're sparse, and nothing at all if you're using trees.**

---

## 12.3 Handling Skewed Distributions: Modern Transformation Methods

Scaling moves and stretches a distribution. It does not change its shape. If your feature is heavily right-skewed—and per Chapter 11, most business quantities are—standardizing it gives you a heavily right-skewed feature with mean zero. For a linear model that assumes roughly symmetric errors, that's the same problem in new units.

Changing the shape needs a nonlinear transformation.

**The log transform** is the workhorse, and it's worth being precise about *why* it works rather than treating it as a magic skew-fixer. A log transform converts multiplicative relationships into additive ones. If your process generates values by compounding—revenue per customer, city populations, file sizes, claim severity, session durations—then the natural structure is multiplicative, the distribution is roughly log-normal, and taking a log doesn't distort the data so much as finally look at it on the axis it was always living on. Use `log1p` (log of 1+x) so that zeros survive.

**Box-Cox** generalizes this into a family parameterized by λ, and estimates the λ that best normalizes your data—λ=0 recovers the log, λ=0.5 is a square root, λ=1 is no transform. It requires strictly positive input, which rules out most real columns.

**Yeo-Johnson** is Box-Cox extended to handle zeros and negatives, and it's the better default for exactly that reason. In scikit-learn both live in `PowerTransformer`, and it will fit λ for you.

Now the part that gets skipped, and it's the part that costs money.

**If you transform your target variable, your predictions come back on the transformed scale, and naively inverting them is biased.** Train a model on `log(y)`, predict, then call `exp()` on the prediction, and you have not estimated the mean of y. You've estimated the *median*. For a right-skewed distribution the mean is above the median, so every single prediction you make is systematically low. This is not a rounding issue: for a log-normal with a moderate variance, the gap is easily 10–20%.

Fixing it requires a retransformation correction—either the analytic form `exp(μ + σ²/2)` if you're willing to assume log-normal residuals, or Duan's smearing estimator, which doesn't require the assumption and is about four lines of code. Pick one. What you cannot do is what most teams do, which is nothing, and then spend a quarter wondering why the model under-forecasts total volume by a consistent 14% while every individual prediction looks defensible.

Note how this connects to Chapter 11: the insurer's problem was a heavy-tailed target destabilizing MSE, and modeling `log(severity)` was one of the correct answers. It's still correct—but it comes with this bias attached, and an actuary summing predicted losses across a portfolio will find the error immediately, because the sum is exactly where a systematic per-row bias becomes visible.

One more caution. Transformations make features harder to explain. "A one-unit increase in log-income is associated with..." is a sentence no business stakeholder has ever wanted to hear, and in regulated settings—credit, insurance, hiring—you may need to hand a regulator a model whose terms are interpretable. Transform for the model's sake, then present on the original scale. That's a reporting problem, not a modeling one, and it's easier to solve than a model that can't fit.

---

## 12.4 Discretization and Binning Strategies

Binning converts a continuous variable into buckets. It's the one technique in this chapter that deliberately throws information away, which means it needs a reason.

**Equal-width** cuts the range into n intervals of the same size. Fast, and useless on skewed data—you'll get one bin with 97% of your rows and nine bins with the tail.

**Equal-frequency (quantile)** cuts so each bin holds the same number of rows. Almost always the better default, because it adapts to the actual distribution.

**K-means binning** uses 1-D clustering to place boundaries where the data is actually sparse, which is a real improvement when the variable is multimodal.

**Supervised / tree-based binning** fits a shallow decision tree against the target and uses its splits as boundaries, so the bins are chosen to be predictive rather than merely tidy. In credit risk this is standard practice, usually expressed as weight-of-evidence binning with an information-value criterion, and it's standard because regulators want a scorecard they can read.

Legitimate reasons to bin: your model is linear and the relationship isn't (binning lets a linear model fit a step function through a curve); the domain has real thresholds (a legal drinking age, a tax bracket, a dosing cutoff); you need a regulator-readable scorecard; or you're deliberately coarsening a feature to make it harder to overfit and easier to keep stable across time.

And the reason not to, which is the one that matters: **binning creates cliffs that don't exist in the world.** Bin age at 65 and your model treats 64 and 65 as categorically different people while treating 65 and 79 as identical. You've imposed a discontinuity on a smooth process, and every row near a boundary gets a prediction that would flip if the underlying value moved by one unit. When someone asks why two nearly-identical customers got different decisions, the answer "they landed on opposite sides of an arbitrary cutoff" is not one you want to give a regulator or a customer.

The blunt version: if you're using a tree-based model, don't bin. Trees find their own splits, optimized against your actual objective, and every one they find will be at least as good as the ones you hand-drew. Pre-binning for a gradient-boosted model is doing the model's job worse than the model would.

---

## 12.5 Polynomial and Interaction Features

Linear models can't represent "the effect of A depends on B." If your process actually works that way—and business processes constantly do—you either use a model that learns interactions natively or you build them by hand.

`PolynomialFeatures(degree=2)` generates every squared term and every pairwise product. It's one line, and here's the arithmetic before you run it: with n input features, degree 2 gives you (n+2)(n+1)/2 outputs. Ten features become 66. A hundred features become 5,151. Five hundred features become 125,751, at which point you have more columns than rows and your model is going to memorize the training set with great enthusiasm.

The generated features are also badly collinear by construction—x and x² move together—which makes linear coefficients unstable and uninterpretable. If you're going to do this, regularize, and scale *before* generating the terms, because squaring a feature that ranges to 45,000 produces one that ranges to two billion.

The alternative that works better in practice is to build the handful of interactions you have a reason to believe in. Price per square foot. Revenue per employee. Debt-to-income. Utilization as a ratio of usage to capacity. Distance divided by time. Every one of these is an interaction that a domain expert would name in a sentence, they're interpretable, they don't explode your feature count, and they routinely outperform a brute-force polynomial expansion because they encode something true rather than something combinatorial. This is Chapter 14's territory and the most valuable thing in it.

Worth knowing: tree ensembles learn interactions natively, but *inefficiently* for multiplicative relationships. A tree approximates `a × b` with a staircase of axis-aligned splits, and it needs depth and data to get there. So handing a gradient-boosted model an explicit ratio feature often helps a lot, even though the model could theoretically discover it. "Theoretically discoverable" and "discoverable with the data you actually have" are different claims.

---

## 12.6 Pipeline Integration and Data Leakage Prevention

Everything in this chapter has fitted parameters. `StandardScaler` learns a mean and a standard deviation. `MinMaxScaler` learns a min and a max. `QuantileTransformer` learns the full empirical distribution. `PowerTransformer` learns λ.

Which produces the rule this chapter exists to enforce, and it's the same rule Chapter 10 gave for imputers: **anything with fitted parameters is part of your model, and it must be fit on training data only.**

```python
# Wrong. The scaler saw the test set.
X_scaled = StandardScaler().fit_transform(X)
X_train, X_test = train_test_split(X_scaled)

# Right.
X_train, X_test = train_test_split(X)
scaler = StandardScaler().fit(X_train)
X_train_s, X_test_s = scaler.transform(X_train), scaler.transform(X_test)
```

The first version leaks. Your test set's mean and variance participated in computing the scaling constants applied to your training data, so your validation score is measuring performance on data your preprocessing had already seen. The inflation is usually small—a point or two—which is precisely why it survives review. It's not large enough to look like a bug. It's just large enough to make you pick the wrong model, ship a system that underperforms its evaluation, and have no idea why.

With `QuantileTransformer` and `PowerTransformer` the leak is considerably worse than with `StandardScaler`, because they learn far more about the distribution's shape.

The structural fix is to stop doing this by hand:

```python
pipe = Pipeline([
    ("impute", SimpleImputer(strategy="median", add_indicator=True)),
    ("scale",  RobustScaler()),
    ("model",  Ridge()),
])
cross_val_score(pipe, X, y, cv=5)
```

Now the imputer and the scaler are refit inside every fold, automatically, and the leak is structurally impossible rather than something you have to remember. `ColumnTransformer` extends this to per-column-type handling. The `Pipeline` object isn't a style preference; it's the mechanism that makes the correct thing the default thing.

Three more traps that a `Pipeline` alone won't save you from.

**Time series.** A random split is already wrong; use `TimeSeriesSplit`. But note that even a correct temporal split leaves the scaler able to leak, because fitting it on the full training window means it learned statistics from the end of that window and applied them to the beginning. For most problems that's acceptable. For anything where the distribution moves materially over time, use an expanding-window fit, and know which one you chose.

**Production.** Your scaler's mean and standard deviation are learned constants. Serialize them with the model, version them together, and never recompute them at serving time from live traffic. A serving path that computes its own scaling statistics from the current batch is Chapter 9's train/serve skew, rebuilt from scratch in a new place, and it will drift the moment your live distribution shifts. If you're using a feature store, the scaler belongs in the transformation layer with everything else.

**Unseen extremes.** `MinMaxScaler` maps to [0,1] on training data. Production sends a value above the training max. Now you have 1.4, outside the range every downstream assumption was built on. Decide explicitly whether you clip, pass it through, or reject the record—and test it, because the default behavior is usually "pass it through silently," which means an out-of-range input reaches your model with no signal that anything unusual happened.

---

## Quick Wins: Scaling Checks Before Your Next Training Run

**Print your feature ranges (5 min).** Run `X.describe().T` and look at the min and max columns. If the widest range is more than a couple orders of magnitude above the narrowest, and your model is distance-based, gradient-descent-trained, regularized, or PCA, you have the freight problem right now. This is a five-minute check that would have saved that marketplace fourteen months.

**Grep for the leak (10 min).** Chapter 10 had you run this check on imputers; run it now on scalers. Search your training code for `fit_transform` and check each hit against the split. Anything fit on everything has already described your test set's distribution to your model.

**Check whether your scaler does anything (15 min).** If your model is a gradient-boosted tree, train it once with your scaling step and once without. The metrics will be materially identical. Delete the step—you're carrying a fitted artifact through training, serialization, and serving for no benefit, and it's one more thing that can desynchronize.

**Check your `Normalizer` imports (2 min).** If `Normalizer` appears anywhere in a tabular pipeline, confirm somebody meant it. In my experience roughly half the time they meant `StandardScaler`.

---

## Your Homework

### Exercise 1: Reproduce the Freight Bug (Time: ~30 minutes)

Take any dataset with features on different scales. Fit a k-NN model on the raw features and again on standardized features, and compare accuracy. Then do the part that actually teaches you something: for a single test point, print the per-feature contribution to the distance for its nearest neighbor. You will see one feature accounting for nearly all of it. That's the number that stayed hidden for fourteen months, and once you've seen the printout you'll never again ship a distance-based model without checking it.

### Exercise 2: Measure the Leak (Time: ~45 minutes)

Build the same model twice: once with the scaler fit on the full dataset before splitting, once with it inside a `Pipeline` and cross-validated properly. Record both validation scores. The gap is your leak, measured in your own data rather than taken on faith from a book. Then do it again with `QuantileTransformer` instead of `StandardScaler` and watch the gap widen. Write both numbers down. The next time someone tells you the leak is too small to bother with, you'll have a figure.

### Exercise 3: Find the Retransformation Bias (Time: ~45 minutes)

Take a right-skewed target. Train a model on `log1p(y)`, predict, and invert with `expm1()`. Now compare the *sum* of your predictions to the sum of the actuals on your test set. It will be low, consistently. Then apply a smearing correction and compare again. The gap between those two totals is a systematic bias that per-row error metrics will never show you, which is why it survives in so many production forecasting systems. Every aggregate anyone builds on top of that model inherits it.

---

## Bridge to Chapter 13

Everything in this chapter assumed your features were numbers—that they could be centered, stretched, squared, and bucketed, because arithmetic on them meant something.

A great deal of your data isn't like that. Product category, country, device type, merchant ID, diagnosis code, job title. You cannot take the mean of a country. And the standard move—assign each category an integer and hand it to the model—recreates Chapter 2's original sin exactly: the model reads those integers as quantities, decides that category 7 is greater than category 3, and starts doing arithmetic on a fact about the world that has no order at all.

Chapter 13 is about encoding: one-hot and why it collapses at high cardinality, target encoding and the leakage it invites through the front door, hashing and embeddings when you have half a million merchant IDs, and the problem that ends more production models than any other item in this part of the book—what your encoder does on a Tuesday when a category shows up that wasn't in the training data.

---

*P.S. — The best part of the freight story is that everyone involved was competent. The ops team built a real safety rating. The data team built a reasonable model. The product team asked for the right thing. Nobody was lazy, nobody cut a corner, and the system still spent fourteen months recommending dangerous carriers because pounds are numerically larger than a five-point scale. There is no code review that catches this, no test that fails, and no alert that fires. There is only somebody, eventually, printing out the distance calculation and going "huh." Be that person early.*
