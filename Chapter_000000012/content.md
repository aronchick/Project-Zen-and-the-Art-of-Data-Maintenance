# Chapter 12: Data Transformation and Scaling

## Or: Your Model Was Measuring Pounds

A freight marketplace moved about 900 shipments a day. Shippers posted loads, carriers bid, and the platform's job was to recommend which carriers to surface for a given load. Get it right and the freight arrives on time and undamaged; get it wrong and you've handed a fragile pallet to the cheapest guy with a truck.

They built a matching model on five features, which is to say five columns it was allowed to look at: distance in miles, weight in pounds, the carrier's on-time percentage, years in business, and a safety rating from 1 to 5 that the operations team had spent a year and a half building. The safety rating was the whole point. It was the thing that made the marketplace worth more than a spreadsheet of phone numbers.

The model used k-nearest-neighbors to find comparable historical matches. k-nearest-neighbors is about the simplest model there is: to make a recommendation for a new load, find the k past loads whose feature values sit closest to it, and go with what worked for them. "Closest" means a distance, and the distance was Euclidean—the straight-line distance from school geometry, extended to five dimensions. It shipped. And for fourteen months it was, by the standard of the metric anyone was watching, fine.

Then the claims data got reviewed properly. Damage and late-delivery claims had reached $4.6 million a year, concentrated in a set of carriers that the safety rating had flagged as poor from the beginning. The model had been recommending them the entire time.

Here's the arithmetic that explains it. Euclidean distance sums the squared differences across features. Weight ranged from about 200 pounds to 45,000. Safety rating ranged from 1 to 5. So when the model compared two candidate matches, a difference of **four pounds** of freight contributed as much to the distance calculation as the entire gap between the safest carrier in the network and the most dangerous one.

The safety rating wasn't underweighted. It was *arithmetically invisible*. Eighteen months of operations work, reduced to rounding error by a unit of measurement. The model was not matching on quality, or reliability, or tenure. It was matching on weight, with a rounding correction for distance, and everything else in that feature vector—the row of five numbers the model actually compares—was decorative.

The fix was one line:

```python
X = StandardScaler().fit_transform(X)
```

On-time delivery went from 82% to 91% in the following quarter. Nobody added a feature, nobody touched the architecture. They just stopped measuring quality in pounds. (That one-liner is the idea, not the code. Section 12.6 has the version that doesn't fit the scaler on the test set.)

---

This chapter's disaster sits up front like all the others, but it's worth naming what makes this one different. Chapters 10 and 11 were about judgment—someone made a defensible call about nulls or outliers and the call was wrong. This one isn't judgment. Nobody decided the safety rating should be ignored. The decision was made by the numeric range of a column, on their behalf, silently, and it held for fourteen months because there is no error message for "your distance metric is in the wrong units."

That's what makes scaling the most boring dangerous topic in this book. It fails without symptoms. The model trains, the metrics compute, the dashboard is green, and the only evidence is a business outcome that nobody thinks to connect back to a preprocessing step.

Two terms carry the rest of this chapter, so here they are in plain English before anything depends on them.

**Feature scaling** is changing the numeric range of a column without changing what it says. Every value in the column goes through the same shift-and-stretch—subtract something, divide by something—so that a column measured in pounds and a column on a one-to-five scale end up in comparable units. The rows keep their order, the distribution keeps its shape, and the only thing that changes is the ruler. That is the whole fix for the freight model: one column was shouting and four were whispering, and scaling turns the volume to the same level on all five.

A **transformation**, as this chapter uses the word, is the broader move: pushing a column through a function—a logarithm, a square root, a bucket assignment—that changes the *shape* of its distribution, not just its range. Scaling can't fix a column whose values pile up near zero and trail off for miles, because stretching a lopsided shape gives you a bigger lopsided shape. Transformation can.

Both happen to the data before the model ever sees it, and both learn something from the data in order to do their job: a mean, a range, a curve parameter. Hold onto that, because the last section of this chapter is entirely about what goes wrong when you forget that what they learned is part of the model.

---

## 12.1 Feature Scaling: Algorithm Requirements and Performance Impact

The first thing to get straight is that "should I scale?" has an actual answer, and it depends entirely on how your algorithm consumes the numbers. There are three groups.

**Algorithms that break without scaling.** Anything that computes a distance: k-nearest-neighbors, k-means, hierarchical clustering, DBSCAN, and SVMs with an RBF kernel. All of them share the cold open's failure mode—the feature with the widest numeric range becomes the metric, and everything else becomes noise. Also in this group, though here it shows up as slow convergence rather than outright failure: anything trained by gradient descent, neural networks included. Gradient descent is the fitting procedure behind neural networks and much of the rest of the field: start with guessed parameters, measure how wrong the model is (the loss), nudge every parameter a small step in the direction that reduces it, and repeat a few thousand times. The size of that step is the learning rate. When features have wildly different scales, the loss surface becomes a long narrow valley, and gradient descent bounces off the walls instead of running down the floor. You'll see it as slow convergence, or as a learning rate that's simultaneously too large for one feature and too small for another, which no amount of tuning fixes because it's not a learning-rate problem.

**Algorithms where scaling changes the answer in a way people miss.** This is the group that costs the most, because the model still *works*—it just works on a different problem than you think.

Regularized regression is the big one. A linear model's prediction is a weighted sum of its features, and the weights are the coefficients. Regularization adds a penalty to the training objective for coefficients that grow large, which is how you stop a linear model from fitting noise; Ridge, Lasso, and ElasticNet are linear regression plus that penalty in three flavors, and every one of them penalizes the *magnitude* of the coefficients. But a coefficient's magnitude depends on its feature's units. Measure a distance in miles and its coefficient is some number; measure the same distance in feet and the coefficient is 5,280 times smaller, so the penalty barely touches it. **You are not regularizing your features, you are regularizing your unit choices.** Run Lasso on unscaled data and the variables it zeroes out are substantially determined by whoever picked the units upstream, which is usually a database schema written by someone who has never heard of your model. This is the most common serious scaling bug in production code, and like the freight disaster it produces a model that looks completely reasonable.

PCA has the same disease for the same reason. Principal component analysis takes a table of features and finds the few directions—each a weighted mix of the original columns—along which the data spreads out most, so you can keep those and drop the rest. It finds directions of maximum variance, and variance is in squared units, so the component structure is dominated by whichever feature has the largest numeric spread. Unscaled PCA on a table with a revenue column returns, essentially, the revenue column with extra steps. (Chapter 15 goes further into this.)

**Algorithms that don't care at all.** Decision trees and every ensemble built on them—random forests, XGBoost, LightGBM, CatBoost. A tree splits on *order*, not magnitude. The rule "weight > 12,000" partitions the data identically whether weight is in pounds, kilograms, or tons; the threshold just changes to match. Any strictly monotonic transformation of a feature—one that never swaps the order of two values—produces the same tree; one that merges values, like rounding or binning, does not. This is why gradient-boosted trees are so forgiving of raw messy tabular data, and it's why a large amount of the scaling code in the world is doing nothing at all.

That last point deserves emphasis, because the reflex runs both directions. About as often as you find a k-NN model measuring pounds, you find a `StandardScaler` sitting in front of an XGBoost model, adding a fitted artifact that has to be versioned, shipped, and kept in sync between training and serving—for zero benefit. It's not harmful to the math. It's harmful to the system, because every fitted preprocessing step is another thing that can drift out of sync between your training and serving paths, which is Chapter 9's entire disaster waiting for a chance.

So the question isn't "should I scale." It's "does my algorithm use magnitude, distance, or a penalty term?" If yes, scaling is mandatory. If no, scaling is a liability you're carrying for aesthetic reasons.

---

## 12.2 Core Scaling Techniques and When to Use Them

Six things people call scalers. They are not interchangeable, and one of them is routinely used by accident. All six come from `sklearn.preprocessing`, and all six share one interface: `.fit(X_train)` learns the constants, `.transform(X)` applies them, and `.fit_transform(X_train)` does both in one call.

**StandardScaler** (`StandardScaler()`, the cold open's one-liner)—subtract the mean, divide by the standard deviation. Output has mean 0 and standard deviation 1, unbounded in both directions. The default, correctly. Use it when your feature is roughly symmetric and you don't have wild outliers. Note what it does *not* do: it does not make anything normally distributed. A bimodal feature—two humps instead of one—standardized is still bimodal, now centered at zero. People say "normalize" and mean this, and then assume normality they didn't get.

**MinMaxScaler** (`MinMaxScaler()`, or `MinMaxScaler(feature_range=(-1, 1))` for a different box)—subtract the min, divide by the range, output bounded to [0, 1]. Useful when you need a hard bound: image pixel values, neural network inputs where a bounded activation expects it. And it is catastrophically fragile, because the min and max are the two most outlier-sensitive statistics in existence. One data-entry error of 9,999,999 in your training set and, if your real values top out around 100, every one of them gets squashed into the bottom 0.001% of the range. You have not scaled your data; you have converted it into a constant with one interesting point. Chapter 11's material is a prerequisite for using this safely.

**RobustScaler** (`RobustScaler()`)—subtract the median, divide by the interquartile range, the spread between the 25th and 75th percentiles. Same idea as StandardScaler, built from statistics that don't care about the tails. When you have outliers you've decided to keep (which, per Chapter 11, is often the right call), this is the scaler that lets you keep them without letting them set the scale for everything else. Underused.

**MaxAbsScaler** (`MaxAbsScaler()`)—divide by the maximum absolute value, output in [-1, 1]. Its one distinguishing property: it doesn't shift the data, so zeros stay zeros and sparse matrices—matrices that are mostly zeros, stored by recording only the nonzero entries—stay sparse. If you're working with high-dimensional sparse data—text features, one-hot encodings at scale—StandardScaler will refuse outright, raising rather than centering a sparse matrix, and the usual workaround of calling `.toarray()` first turns a 2 GB sparse matrix into a 400 GB dense one. This is a real way to run a machine out of memory. (`StandardScaler(with_mean=False)` is the other workaround: it skips the centering and keeps the matrix sparse, at which point it is doing what `MaxAbsScaler` does with a different divisor.)

**Normalizer**—and this is the one people use by accident. Every other scaler on this list works down a *column*, one feature at a time, fit across your rows. `Normalizer()` works across a *row*, scaling each sample to unit norm—dividing the row by its own length so that every row ends up with length 1. It's a completely different operation for a completely different purpose (comparing direction rather than magnitude, standard in text similarity). It is next to `StandardScaler` in the docs, it sounds like the generic option, and if you drop it into a tabular pipeline it will silently erase every feature's magnitude and leave only each row's direction. Check your imports.

**QuantileTransformer** (`QuantileTransformer(output_distribution="normal")`; the default output is uniform)—map each value to its rank among a thousand quantile landmarks, then to a uniform or normal distribution. This is the heavy artillery: it's immune to outliers by construction, and it will force any input distribution into the output shape you asked for, no matter how ugly the input was. The costs are real. It discards the *shape* of the distribution and keeps only the ordering, so the distance between two points stops meaning what it used to mean. It needs enough data to estimate quantiles reliably. And it will happily map a value it never saw during training onto the edge of the distribution, which makes its production behavior on new extremes worth testing deliberately.

The practical default: **RobustScaler if you have outliers, StandardScaler if you don't, MaxAbsScaler if you're sparse, and nothing at all if you're using trees.**

---

## 12.3 Handling Skewed Distributions: Modern Transformation Methods

Scaling moves and stretches a distribution. It does not change its shape. If your feature is heavily right-skewed—and per Chapter 11, most business quantities are—standardizing it gives you a heavily right-skewed feature with mean zero. The skew is untouched, so for a linear model it's the same leverage problem in new units.

Changing the shape needs a nonlinear transformation.

**The log transform** is the workhorse, and it's worth being precise about *why* it works rather than treating it as a magic skew-fixer. A log transform converts multiplicative relationships into additive ones. If your process generates values by compounding—revenue per customer, city populations, file sizes, claim severity, session durations—then the natural structure is multiplicative, the distribution is roughly log-normal (meaning the *logarithm* of the values is bell-shaped, even though the values themselves are not), and taking a log doesn't distort the data so much as finally look at it on the axis it was always living on. Use `np.log1p(x)` (log of 1+x) so that zeros survive; `np.expm1()` is its inverse.

**Box-Cox** generalizes this into a family parameterized by λ; `PowerTransformer(method="box-cox")` estimates the λ that best normalizes your data—λ=0 recovers the log, λ=0.5 is a square root, λ=1 is no transform. It requires strictly positive input, which rules out any column carrying a zero or a negative.

**Yeo-Johnson** is Box-Cox extended to handle zeros and negatives: `PowerTransformer(method="yeo-johnson")`, which is also scikit-learn's default, so a bare `PowerTransformer()` gets you the same thing. It's the better default for exactly that reason. Either way the fitted λ is there to read back in `.lambdas_`, and either way the output is standardized by default too, so it is a transform and a scaler in one step.

Now the part that gets skipped, and it's the part that costs money.

**If you transform your target variable, your predictions come back on the transformed scale, and naively inverting them is biased.** The target is the column you're trying to predict—the `y` in every `fit(X, y)`—and everything above was about the features, the `X`. Train a model on `np.log(y)`, predict, then call `np.exp()` on the prediction, and you have not estimated the mean of y. You've estimated the *median*. For a right-skewed distribution the mean is above the median, so every single prediction you make is systematically low. This is not a rounding issue: for a log-normal with a moderate variance, the gap is easily 10–20%.

Fixing it requires a retransformation correction. The analytic form is `exp(μ + σ²/2)`, where σ² is the variance of the residuals on the log scale—`np.exp(pred_log + resid_log.var() / 2)`—and it's right if you're willing to assume log-normal residuals. Duan's smearing estimator doesn't require the assumption and is about four lines of code:

```python
model.fit(X_train, np.log(y_train))
resid = np.log(y_train) - model.predict(X_train)
smear = np.mean(np.exp(resid))                 # Duan's smearing factor
y_pred = np.exp(model.predict(X_test)) * smear
```

Pick one. What you cannot do is what most teams do, which is nothing, and then spend a quarter wondering why the model under-forecasts total volume by a consistent 14% while every individual prediction looks defensible.

Note how this connects to Chapter 11: the insurer's problem was a heavy-tailed target destabilizing MSE (mean squared error, the usual training objective for a regression), and modeling `log(severity)` was one of the correct answers. It's still correct—but it comes with this bias attached, and an actuary summing predicted losses across a portfolio will find the error immediately, because the sum is exactly where a systematic per-row bias becomes visible.

One more caution. Transformations make features harder to explain. "A one-unit increase in log-income is associated with..." is a sentence no business stakeholder has ever wanted to hear, and in regulated settings—credit, insurance, hiring—you may need to hand a regulator a model whose terms are interpretable. Transform for the model's sake, then present on the original scale. That's a reporting problem, not a modeling one, and it's easier to solve than a model that can't fit.

---

## 12.4 Discretization and Binning Strategies

Binning converts a continuous variable into buckets. It's the technique in this chapter that throws the most information away on purpose, which means it needs a reason. In scikit-learn the general tool is `KBinsDiscretizer`, whose `strategy` argument picks among the first three methods below; pandas has `pd.cut` and `pd.qcut` for the first two.

**Equal-width** cuts the range into n intervals of the same size: `pd.cut(x, bins=10)`, or `KBinsDiscretizer(n_bins=10, encode="ordinal", strategy="uniform")` if you want integer bin ids from a scikit-learn step. Fast, and useless on skewed data—you'll get one bin with 97% of your rows and nine bins with the tail.

**Equal-frequency (quantile)** cuts so each bin holds the same number of rows: `pd.qcut(x, q=10)`, or `strategy="quantile"`, which is `KBinsDiscretizer`'s default. Almost always the better default, because it adapts to the actual distribution.

**K-means binning** uses 1-D clustering (`strategy="kmeans"`) to place boundaries where the data is actually sparse, which is a real improvement when the variable is multimodal—several distinct humps rather than one.

**Supervised / tree-based binning** fits a shallow decision tree against the target and uses its splits as boundaries, so the bins are chosen to be predictive rather than merely tidy:

```python
tree = DecisionTreeClassifier(max_leaf_nodes=5).fit(x.reshape(-1, 1), y)
cuts = np.sort(tree.tree_.threshold[tree.tree_.feature >= 0])
```

(`DecisionTreeRegressor` if the target is a number rather than a class.) In credit risk this is standard practice, usually expressed as weight-of-evidence binning with an information-value criterion—each bin gets a score for how sharply it separates the good outcomes from the bad, and the boundaries are chosen to maximize the total—and it's standard because regulators want a scorecard they can read. The `optbinning` library does that optimization: `OptimalBinning(dtype="numerical").fit(x, y).splits`.

Legitimate reasons to bin: your model is linear and the relationship isn't (binning lets a linear model fit a step function through a curve); the domain has real thresholds (a legal drinking age, a tax bracket, a dosing cutoff); you need a regulator-readable scorecard; or you're deliberately coarsening a feature to make it harder to overfit and easier to keep stable across time.

And the reason not to, which is the one that matters: **binning creates cliffs that don't exist in the world.** Bin age at 65 and your model treats 64 and 65 as categorically different people while treating 65 and 79 as identical. You've imposed a discontinuity on a smooth process, and every row near a boundary gets a prediction that would flip if the underlying value moved by one unit. When someone asks why two nearly-identical customers got different decisions, the answer "they landed on opposite sides of an arbitrary cutoff" is not one you want to give a regulator or a customer.

The blunt version: if you're using a tree-based model, you almost never need to bin. Trees find their own splits, optimized against your actual objective, and LightGBM, XGBoost's `hist` mode and `HistGradientBoosting` already bin every feature for you—255 buckets by default, 256 in XGBoost. Hand-binning ahead of a gradient-boosted model is usually redundant work.

---

## 12.5 Polynomial and Interaction Features

Linear models can't represent "the effect of A depends on B." If your process actually works that way—and business processes constantly do—you either use a model that learns interactions natively or you build them by hand.

`PolynomialFeatures(degree=2)` generates every squared term and every pairwise product. It's one line, and here's the arithmetic before you run it: with n input features, degree 2 gives you (n+2)(n+1)/2 outputs, counting the originals and the bias column—a column of ones, there to give the model an intercept—it adds by default. Ten features become 66. A hundred features become 5,151. Five hundred features become 125,751, at which point you have more columns than rows and your model is going to memorize the training set with great enthusiasm. `PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)` keeps the originals and the pairwise products and drops the squares and the bias column, which is usually what people actually wanted; it trims those counts to 55, 5,050 and 125,250, which is still a lot.

The generated features are also badly collinear by construction—x and x² move together—which makes linear coefficients unstable and uninterpretable. If you're going to do this, regularize, and scale *before* generating the terms, because squaring a feature that ranges to 45,000 produces one that ranges to two billion.

The alternative that works better in practice is to build the handful of interactions you have a reason to believe in. Price per square foot. Revenue per employee. Debt-to-income. Utilization as a ratio of usage to capacity. Distance divided by time. Every one of these is one line of pandas—`df["price_per_sqft"] = df["price"] / df["sqft"]`—and an interaction that a domain expert would name in a sentence. They're interpretable, they don't explode your feature count, and they routinely outperform a brute-force polynomial expansion because they encode something true rather than something combinatorial. This is Chapter 14's territory and the most valuable thing in it.

Worth knowing: tree ensembles learn interactions natively, but *inefficiently* for multiplicative relationships. A tree approximates `a × b` with a staircase of axis-aligned splits, and it needs depth and data to get there. So handing a gradient-boosted model an explicit ratio feature often helps a lot, even though the model could theoretically discover it. "Theoretically discoverable" and "discoverable with the data you actually have" are different claims.

---

## 12.6 Pipeline Integration and Data Leakage Prevention

Everything in this chapter has fitted parameters. `StandardScaler` learns a mean and a standard deviation. `MinMaxScaler` learns a min and a max. `QuantileTransformer` learns a thousand quantile landmarks, by default from a subsample. `PowerTransformer` learns λ.

Which produces the rule this chapter exists to enforce, and it's the same rule Chapter 10 gave for imputers: **anything with fitted parameters is part of your model, and it must be fit on training data only.**

Breaking that rule is data leakage: information from the data you'll evaluate on—or from the future—reaching the model during training, so that the score you compute measures something other than performance on genuinely new data.

```python
# Wrong. The scaler saw the test set.
X_scaled = StandardScaler().fit_transform(X)
X_train, X_test = train_test_split(X_scaled)

# Right.
X_train, X_test = train_test_split(X)
scaler = StandardScaler().fit(X_train)
X_train_s, X_test_s = scaler.transform(X_train), scaler.transform(X_test)
```

The first version leaks. Your test set's mean and variance participated in computing the scaling constants applied to your training data, so your validation score is measuring performance on data your preprocessing had already seen. The inflation is usually small, often well under a point, which is precisely why it survives review. It's not large enough to look like a bug. It's just large enough to make you pick the wrong model, ship a system that underperforms its evaluation, and have no idea why.

With `QuantileTransformer` and `PowerTransformer` there is more to leak than with `StandardScaler`, because they learn far more about the distribution's shape.

The structural fix is to stop doing this by hand:

```python
pipe = Pipeline([
    ("impute", SimpleImputer(strategy="median", add_indicator=True)),
    ("scale",  RobustScaler()),
    ("model",  Ridge()),
])
cross_val_score(pipe, X, y, cv=5)
```

`cross_val_score` is cross-validation: split the data into five folds, train on four and score on the fifth, rotate, and report all five scores. Now the imputer and the scaler are refit inside every fold, automatically, and the leak is structurally impossible rather than something you have to remember. `ColumnTransformer` extends this to per-column-type handling—`ColumnTransformer([("num", RobustScaler(), numeric_cols)], remainder="passthrough")` scales the columns you name and leaves the rest alone—and slots into the pipeline as its first step. The `Pipeline` object isn't a style preference; it's the mechanism that makes the correct thing the default thing.

Three more traps that a `Pipeline` alone won't save you from.

**Time series.** A random split is already wrong; use `TimeSeriesSplit`—`cross_val_score(pipe, X, y, cv=TimeSeriesSplit(n_splits=5))`—which only ever scores on rows later than the ones it trained on. But note that even a correct temporal split leaves the scaler looking into its own future, because fitting it on the full training window means it learned statistics from the end of that window and applied them to the beginning. For most problems that's acceptable. For anything where the distribution moves materially over time, use an expanding-window fit, and know which one you chose.

**Production.** Your scaler's mean and standard deviation are learned constants. Serialize them with the model—`joblib.dump(pipe, "freight-match-v12.joblib")` writes the whole fitted pipeline, scaler constants included—version them together, and never recompute them at serving time from live traffic. A serving path that computes its own scaling statistics from the current batch is Chapter 9's train/serve skew—the training-time and serving-time versions of a feature quietly diverging—rebuilt from scratch in a new place, and it will drift the moment your live distribution shifts. If you're using a feature store, the shared service that computes features once and serves the same values to training and production, the scaler belongs in the transformation layer with everything else.

**Unseen extremes.** `MinMaxScaler` maps to [0,1] on training data. Production sends a value above the training max. Now you have 1.4, outside the range every downstream assumption was built on. Decide explicitly whether you clip (`MinMaxScaler(clip=True)` pins anything outside the training range to 0 or 1), pass it through, or reject the record—and test it, because the default behavior is usually "pass it through silently," which means an out-of-range input reaches your model with no signal that anything unusual happened.

---

## Quick Wins: Scaling Checks Before Your Next Training Run

**Print your feature ranges (5 min).** Run `X.describe().T` and look at the min and max columns. If the widest range is more than a couple orders of magnitude above the narrowest, and your model is distance-based, gradient-descent-trained, regularized, or PCA, you have the freight problem right now. This is a five-minute check that would have saved that marketplace fourteen months.

**Grep for the leak (10 min).** Chapter 10 had you run this check on imputers; run it now on scalers. Search your training code for `fit_transform` and check each hit against the split. Anything fit on everything has already described your test set's distribution to your model.

**Check whether your scaler does anything (15 min).** If your model is a gradient-boosted tree, train it once with your scaling step and once without. The metrics will be materially identical. Delete the step—you're carrying a fitted artifact through training, serialization, and serving for no benefit, and it's one more thing that can desynchronize.

**Check your `Normalizer` imports (2 min).** If `Normalizer` appears anywhere in a tabular pipeline, confirm somebody meant it. Roughly half the time, they meant `StandardScaler`.

---

## Your Homework

### Exercise 1: Reproduce the Freight Bug (Time: ~30 minutes)

Take any dataset with features on different scales. Fit a k-NN model (`KNeighborsClassifier(n_neighbors=5)`) on the raw features and again on standardized features, and compare accuracy. Then do the part that actually teaches you something: for a single test point, print the per-feature contribution to the distance for its nearest neighbor—`knn.kneighbors(x.reshape(1, -1))` finds the neighbor, and `(x - x_neighbor) ** 2` is the contribution, one number per feature. You will see one feature accounting for nearly all of it. That's the number that stayed hidden for fourteen months, and once you've seen the printout you'll never again ship a distance-based model without checking it.

### Exercise 2: Measure the Leak (Time: ~45 minutes)

Build the same model twice: once with the scaler fit on the full dataset before splitting, once with it inside a `Pipeline` and cross-validated properly. Record both validation scores. The gap is your leak, measured in your own data rather than taken on faith from a book. Then do it again with `QuantileTransformer` instead of `StandardScaler` and see whether the gap widens. Write both numbers down. The next time someone tells you the leak is too small to bother with, you'll have a figure.

### Exercise 3: Find the Retransformation Bias (Time: ~45 minutes)

Take a right-skewed target. Train a model on `np.log1p(y)`, predict, and invert with `np.expm1()`. Now compare the *sum* of your predictions to the sum of the actuals on your test set. It will be low, consistently. Then apply a smearing correction and compare again (on the `log1p` scale the factor multiplies `np.exp(pred)`, and the 1 comes off afterwards). The gap between those two totals is a systematic bias that per-row error metrics will never show you, which is why it survives in so many production forecasting systems. Every aggregate anyone builds on top of that model inherits it.

---

## Bridge to Chapter 13

Everything in this chapter assumed your features were numbers—that they could be centered, stretched, squared, and bucketed, because arithmetic on them meant something.

A great deal of your data isn't like that. Product category, country, device type, merchant ID, diagnosis code, job title. You cannot take the mean of a country. And the standard move—assign each category an integer and hand it to the model—recreates Chapter 2's original sin exactly: the model reads those integers as quantities, decides that category 7 is greater than category 3, and starts doing arithmetic on a fact about the world that has no order at all.

Chapter 13 is about encoding: one-hot and why it collapses at high cardinality, target encoding and the leakage it invites through the front door, hashing and embeddings when you have half a million merchant IDs, and the problem that ends more production models than any other item in this part of the book—what your encoder does on a Tuesday when a category shows up that wasn't in the training data.

---

*P.S. — The best part of the freight story is that everyone involved was competent. The ops team built a real safety rating. The data team built a reasonable model. The product team asked for the right thing. Nobody was lazy, nobody cut a corner, and the system still spent fourteen months recommending dangerous carriers because pounds are numerically larger than a five-point scale. There is no code review that catches this, no test that fails, and no alert that fires. There is only somebody, eventually, printing out the distance calculation and going "huh." Be that person early.*
