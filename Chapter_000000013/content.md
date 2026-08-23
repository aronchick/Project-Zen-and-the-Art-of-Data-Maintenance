# Chapter 13: Encoding Strategies for Categorical Variables

## Or: The Model Memorized the Answer Key

A demand-side platform bid on display ad impressions in real time. Roughly $180 million a year of media spend flowed through it, and the platform took a cut, so the whole business rested on one prediction made in under ten milliseconds: given this impression, how likely is a click? Bid accordingly.

The most predictive thing about an impression is where it appears. Not the site—the *placement*: a specific ad slot, on a specific page template, on a specific property. The training data had 2.3 million distinct `placement_id` values.

One-hot encoding 2.3 million categories was obviously off the table, so an engineer reached for target encoding. Replace each placement with the average click-through rate observed for that placement. One column instead of 2.3 million. Directly, obviously informative.

```python
means = train.groupby("placement_id")["clicked"].mean()
train["placement_ctr"] = train["placement_id"].map(means)
```

Offline AUC went from 0.71 to 0.91 on the same model, same split, same everything. That is not an improvement anyone questions on a Friday. They shipped it.

Here is what those three lines actually built. Sixty percent of those 2.3 million placements appeared fewer than five times in the training window. For a placement that appeared exactly once, its "average CTR" is that single row's label. Not correlated with the label. *Is* the label, written into the feature column, sitting right next to it at training time.

The model did what any competent model does when you hand it the answer key: it learned to read the answer key. `placement_ctr == 1.0` means clicked. `placement_ctr == 0.0` means not clicked. It scored 0.91 because it was being graded on a test it had already been given.

In production, none of that transfers. A live impression on a rare placement gets whatever mean was computed from its handful of historical rows, which is a number like 1.0 or 0.5 derived from two observations, and the model treats that number with the same total confidence it learned in training. So the bidder paid up—hard—for inventory it was certain about and wrong about. Live AUC was 0.62. Against a 0.71 baseline the model had replaced.

Eleven weeks and $8.7 million in media spend later, an analyst plotted predicted CTR against realized CTR bucketed by how many times each placement had been seen before, and the whole thing fell out of the chart in about four minutes.

The fix took a day. Compute the encoding out-of-fold so a row never contributes to its own feature. Smooth every category's average toward the global average in proportion to how little evidence supports it. Bucket everything below a frequency floor into an explicit rare category. Offline AUC dropped to 0.78. Live AUC came in at 0.77.

The number on the slide went down. The business went up. That gap—0.91 offline versus 0.78 offline—was never performance. It was the size of the lie.

---

A categorical variable is a fact about the world that does not support arithmetic. There is no mean of a country, no square root of a diagnosis code, no sensible answer to "what is `merchant_id` 4471 minus `merchant_id` 3902."

Every model you will ever train, on the other hand, runs on arithmetic. So encoding is the act of *inventing* arithmetic for something that doesn't have any, and every encoding scheme is a claim about the structure of the thing you're encoding. One-hot claims the categories are mutually exclusive and unrelated. Ordinal claims they have an order. Target encoding claims a category's history predicts its future. Embeddings claim there is latent structure worth learning.

Get the claim right and the model gets a real head start. Get it wrong and you have done what Chapter 2 warned about, one layer deeper: you've told the model something false about the world, in a form it has no way to question.

---

## 13.1 Understanding Categorical Types: Nominal, Ordinal, and Cyclical

Before you pick an encoder, you have to know which of three things you're holding. This is not a property of the column. It's a claim about the world that the column describes, and the column will not tell you which one is true.

**Nominal** categories have no order at all. Country. Merchant. Device type. Diagnosis code. Product category. The only relationship between two nominal values is same-or-different. Any encoding that implies more than that is inventing a fact.

**Ordinal** categories have a real order but no guaranteed spacing. Small, medium, large. Credit ratings from AAA down to D. Education level. A five-point pain scale. The order is genuine—large really is more than medium—but the gaps are not equal and are usually unknowable. When you map small/medium/large to 1/2/3, you have not just encoded the order. You have asserted that the distance from small to medium equals the distance from medium to large, and that going from small to large is exactly twice the step from small to medium. Sometimes that's harmless. For garment sizes it's roughly true. For credit ratings it is spectacularly false: the distance from AAA to AA is nothing like the distance from BBB to BB, which is the exact boundary where institutional mandates force selling.

If you know the real spacing, encode the real spacing. Map credit ratings to historical default rates. Map T-shirt sizes to chest measurements. You are allowed to put domain knowledge into the numbers; that's the whole point of Chapter 14.

**Cyclical** categories have an order that wraps. Month, day of week, hour, compass bearing, wind direction, angle. December is month 12 and January is month 1, and every linear encoding you can write says they are eleven units apart when they are in fact adjacent. Same for hour 23 and hour 0, which are the two halves of one night.

The standard fix is to project onto a circle:

```python
df["hour_sin"] = np.sin(2 * np.pi * df["hour"] / 24)
df["hour_cos"] = np.cos(2 * np.pi * df["hour"] / 24)
```

You need both. Sine alone maps 3 a.m. and 9 p.m. to nearly the same value, which just moves the collision somewhere less obvious. The pair together gives every hour a unique position on a circle where hour 23 and hour 0 sit next to each other, which is where they belong.

One caveat worth internalizing: tree-based models handle cyclical variables tolerably well without this, because a tree can carve out `hour >= 22 OR hour <= 5` given enough depth. Linear models and distance-based models cannot, ever. If you're running k-NN or a linear model on raw hour-of-day, your model believes midnight and 11 p.m. are maximally different, and it will keep believing that no matter how much data you give it.

---

## 13.2 Basic to Advanced Encoding Techniques

**One-hot encoding** is the correct default and you should feel no shame about it. One binary column per category, exactly one of them hot per row. It makes the minimum possible claim—these things are distinct—and it works with everything.

Its cost is width. Fifteen categories is fifteen columns and nobody notices. Fifteen thousand categories is fifteen thousand mostly-zero columns, and now you have a sparse matrix, a memory problem, and a model that has to estimate fifteen thousand coefficients from data that gives it three examples of most of them.

The `drop_first` question comes up constantly and the answer depends entirely on the model. For unregularized linear models, keeping all levels plus an intercept gives you perfect multicollinearity and the fit becomes unstable. Drop one, and it becomes the reference level everything else is measured against. For regularized linear models it matters less but changes what the penalty means—dropped levels get their effect absorbed into the intercept and escape shrinkage. For trees, dropping a level is mildly harmful: it makes one category expressible only as the absence of all the others, which costs the tree extra depth to say.

**Ordinal encoding**—map each category to an integer—is right when the thing is truly ordinal and you've thought about spacing, and wrong the rest of the time. There is a common defense that it's fine for tree models because a tree can recover any grouping through repeated splits. This is true and still not free: with an arbitrary integer assignment, expressing "categories 3, 7, and 12 behave alike" costs several splits and several levels of depth, and every split spent on reconstruction is a split not spent on signal.

**Frequency and count encoding** replaces each category with how often it appears. Cheap, one column, no target involvement so no leakage risk, and often more predictive than it has any right to be—for merchants, sellers, and users, "how much do we see this thing" carries real information about size, tenure, and legitimacy. The obvious failure is collision: two categories with identical counts become identical to the model. Usually acceptable, occasionally not.

**Binary and BaseN encoding** compress cardinality into log-scaled columns by writing the category index in base 2 or base N. It was a reasonable middle ground when memory was tight. The bit patterns are arbitrary, so the model has to learn to reassemble a meaning from digits that have none, and in practice hashing or embeddings do the same job better. Know it exists; reach for it rarely.

**Native categorical handling** is what you should actually be doing on tree models in 2026. LightGBM and CatBoost both accept categorical columns directly and split on them properly—partitioning the category set rather than testing one value at a time. This is strictly better than one-hot for trees at moderate-to-high cardinality, it's one parameter, and a startling number of teams don't know it's there.

```python
import lightgbm as lgb

for c in cat_cols:
    df[c] = df[c].astype("category")

model = lgb.LGBMClassifier()
model.fit(X_train, y_train, categorical_feature=cat_cols)
```

That is the whole intervention. Before you build anything more sophisticated, try this and write down the number.

---

## 13.3 Target-Based Encoding and Regularization

Target encoding replaces a category with a statistic of the target computed over that category: mean CTR per placement, default rate per ZIP, average basket size per store. It is genuinely powerful. It collapses any cardinality to a single dense column, and that column is directly aimed at the thing you're predicting.

It is also the most reliable way there is to leak your labels into your features, and the leak is invisible in every offline metric you have.

You already saw the mechanism. Compute a category's mean over the full training set, and every row in that category has contributed its own label to its own feature value. For a category with ten thousand rows, that contribution is one part in ten thousand and nobody cares. For a category with three rows, the feature is a third of the answer. And in every real high-cardinality dataset, the distribution is long-tailed, so most of your categories live in the regime where the leak dominates.

Two defenses, and you want both.

**Out-of-fold computation.** Split the training data into folds. For rows in fold *k*, compute the encoding using only the other folds. A row's own label is then structurally incapable of reaching its own feature. This is the same discipline as Chapter 9's point-in-time correctness, wearing different clothes: never let a row see information it wouldn't have had.

**Smoothing toward the prior.** Even out-of-fold, a category with two observations produces a noisy estimate that the model has no way to discount. So shrink every category's mean toward the global mean, weighted by how much evidence backs it:

```python
smoothed = (counts * category_mean + m * global_mean) / (counts + m)
```

Read `m` as "how many observations of evidence I demand before I start believing a category's own average." At `m = 20`, a category with 2 rows lands almost entirely on the global mean, a category with 20 rows sits halfway, and a category with 5,000 rows is essentially its own average. Tune `m`; it matters more than most hyperparameters people do tune.

CatBoost's ordered target statistics are the most rigorous version of this idea: it draws a random permutation of the data and encodes each row using only rows that came before it, which is the streaming-safe formulation of out-of-fold. If you're already using CatBoost, you're getting this for free and should not hand-roll a replacement.

Scikit-learn shipped a `TargetEncoder` that does the cross-fitting internally, which removed most people's excuse for the naive version:

```python
from sklearn.preprocessing import TargetEncoder
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer

pre = ColumnTransformer([
    ("cat", TargetEncoder(smooth="auto"), cat_cols),
    ("num", StandardScaler(), num_cols),
])
pipe = Pipeline([("pre", pre), ("model", HistGradientBoostingClassifier())])
cross_val_score(pipe, X, y, cv=5)
```

The `Pipeline` is not decoration. It is the thing that guarantees the encoder is fit inside each CV fold rather than once over everything, which is the exact leak Chapter 12 covered for scalers and which is far more dangerous here—a scaler leaks distribution shape, a target encoder leaks the answer.

One trap worth naming: **leave-one-out encoding**, which computes each row's encoding from every other row in its category, sounds like the strictest possible fix and is in fact worse than smoothed out-of-fold. It leaks in a subtler direction—the encoded value becomes systematically *anti*-correlated with the row's own label, which a model can learn to invert. If it seems too clever, it is.

---

## 13.4 High Cardinality Solutions: Hashing and Entity Embeddings

High cardinality shows up wherever your data contains identifiers rather than descriptions: merchant IDs, user IDs, SKUs, placements, ZIP codes, ICD-10 codes, IP blocks. Hundreds of thousands to millions of levels, a long tail where most levels appear a handful of times, and new levels arriving every day.

**Rare-category bucketing** is the first thing to try and the most underused tool in this chapter. Take everything below a frequency floor and call it `__RARE__`. You typically lose nothing—a category seen twice supports no reliable estimate anyway—and you convert an unbounded vocabulary into a bounded one, which fixes memory, fixes the noisy-estimate problem, and gives you a natural home for unseen categories at serving time. Scikit-learn's `OneHotEncoder` has this built in via `min_frequency` and `max_categories`. Set them.

**The hashing trick** maps each category through a hash function into a fixed number of buckets. Fixed memory regardless of cardinality, no vocabulary to store or ship, no training-time knowledge of the category set required, and unseen categories are handled automatically because a hash function has an opinion about every possible string.

```python
from sklearn.feature_extraction import FeatureHasher

hasher = FeatureHasher(n_features=2**18, input_type="string")
X_hashed = hasher.transform(df["merchant_id"].astype(str).apply(lambda s: [s]))
```

The cost is collisions—two unrelated merchants sharing a bucket—and total loss of interpretability, since you cannot invert a hash to ask which category a coefficient belongs to. Collisions bother people more than they should. A collision only hurts when both colliding categories are individually informative *and* point in opposite directions, and with buckets set to several times your effective cardinality that intersection is small. Size the space generously; powers of two, and more than you think.

**Entity embeddings** learn a dense vector per category as part of model training, the way word embeddings work in language models. The technique got its reputation from a Kaggle solution to Rossmann store sales forecasting, where embedding store IDs beat carefully hand-engineered store features. A common starting rule for dimension is `min(50, (cardinality + 1) // 2)`, which nobody should treat as more than a starting point.

What you get beyond accuracy is a similarity space, and that is often the more valuable artifact. After training, stores that behave alike sit near each other in embedding space, whether or not anyone ever labeled them as similar. Merchandising teams will use that map long after the model that produced it is retired.

What you pay is real: you need a neural network or a library that supports embedding layers, you need enough data per category for the vectors to mean anything, and you have added a component that requires its own versioning and its own explanation to whoever audits your model. Embeddings are worth it when cardinality is high, data is plentiful, and you have reason to believe the categories have latent structure. They are not worth it because they sound better in a review.

---

## 13.5 Handling Unknown Categories in Production

This section ends more production models than anything else in this part of the book, and it gets about a paragraph in most treatments of encoding.

Your encoder was fit on a fixed vocabulary. Production is not fixed. A new merchant onboards. A vendor adds a device type. Someone in operations creates a status code on a Tuesday afternoon to unblock a customer. Now a value arrives that the encoder has never seen, and one of three things happens.

**It raises.** `OneHotEncoder` with default settings throws on unknown input. Your inference service returns a 500. This is the *good* outcome, because you find out.

**It silently zeroes.** With `handle_unknown="ignore"`, an unknown value produces a row of all zeros. That is not a neutral outcome; it is the encoder asserting "this record belongs to none of the known categories," which for a linear model means the entire categorical contribution vanishes and the prediction shifts toward the intercept. Every unknown value gets the same silent, confident, wrong answer. `handle_unknown="infrequent_if_exist"` is usually what you actually wanted—it routes unknowns into the infrequent bucket you already trained on, which at least has a learned response.

**It gets the global mean.** Target encoders typically fall back to the prior for unseen categories, which is defensible and still needs to be a decision you made rather than a default you inherited.

The real fix is to train the model on unknowns so it has a learned response to them. You cannot expect sensible behavior on a case you never showed it. The technique is to reserve an explicit `__UNKNOWN__` category and, during training, randomly reassign a small fraction of rare-category rows into it. The model learns what to do when it doesn't recognize something, which is exactly the skill you need it to have on Tuesday.

Then monitor it. **Unknown rate per categorical feature belongs in your monitoring dashboard next to your accuracy metrics** (Chapter 25 covers the wider practice). A feature that runs at 0.3% unknowns for a year and jumps to 12% overnight is not a modeling event. It is an upstream schema change nobody told you about, and it is exactly what the data contracts in Chapter 5 exist to prevent.

Two variants of this failure are worth naming separately because monitoring the unknown rate will not catch either one.

**Category drift with a stable vocabulary.** The string `PENDING` was in your training data and is still arriving, but after a product launch it means something different than it used to. No unknown, no error, no alert. Your encoder maps it to the same coefficient it learned from the old meaning. Only a contract or a human catches this.

**Encoding skew between training and serving.** Training encoded `merchant_category` from the warehouse, where it is title case and trimmed. Serving reads the same conceptual field from an upstream service where it arrives as `"restaurant "`, lowercase with a trailing space. Different string, therefore unknown category, therefore silent zeros on a feature that mattered. This is Chapter 9's train/serve skew arriving through the encoder rather than the feature pipeline, and the defense is the same: one definition, computed once, shared by both paths.

---

## 13.6 Encoding Decision Matrix and Best Practices

| Situation | Start with | The thing that bites |
|---|---|---|
| Under ~15 levels, any model | One-hot | Nothing much. Move on. |
| Under ~15, unregularized linear | One-hot, drop one level | Multicollinearity if you keep all |
| Genuine order, known spacing | Map to the real quantity | Assuming the gaps are equal |
| Genuine order, unknown spacing | Ordinal, then check | Treating 1/2/3 as arithmetic |
| Cyclical | sin/cos pair | Shipping only the sine |
| 15–1,000 levels, tree model | Native categorical support | Not knowing it exists |
| 15–1,000 levels, linear model | Target encoding, OOF + smoothed | Fitting the encoder outside the CV fold |
| Over 1,000, need it this week | Rare bucketing, then hashing | Undersizing the hash space |
| Over 1,000, deep model, lots of data | Entity embeddings | Building it because it sounds good |
| Anything, in production | All of the above, plus explicit unknown handling | The Tuesday problem |

The practices worth writing on the wall:

**Fit encoders inside the pipeline, always.** If your encoder is fit before the train/test split, your validation score is fiction. This is the same rule as Chapter 12 and it is more expensive to break here.

**Set a frequency floor before you do anything clever.** Rare-category bucketing solves a surprising fraction of high-cardinality problems on its own, and it makes every downstream encoder better behaved.

**Try native categorical handling before you engineer.** One parameter on LightGBM or CatBoost, measured properly, is the baseline that a lot of elaborate encoding work fails to beat.

**Write down the unknown-category behavior for every categorical feature you ship.** Not the code—the behavior, in a sentence, in a document a human can read during an incident. "Unseen merchant categories route to `__RARE__`, which was trained on 4% of rows and behaves like a slightly-below-average merchant." If you cannot write that sentence today, you do not know what your model does on Tuesday.

**Version the encoder with the model.** An encoder is a learned artifact with a fitted vocabulary, exactly as much a part of your model as the weights. Deploying a model against an encoder fitted on a different vocabulary is a silent category shift on every single feature, and it happens more often than anyone admits.

---

## Quick Wins: Four Ways to Break Your Encoder on Purpose

**Send it a category that doesn't exist (10 min).** In staging, post a prediction request with `merchant_category = "__zzz_not_a_real_thing"`. Record exactly what comes back: an exception, a prediction, or a prediction that looks fine and isn't. Do this for every categorical feature you serve. This is the single highest-value ten minutes in this chapter.

**Count your tail (10 min).** For each categorical column, print what fraction of levels account for 95% of rows. If ten levels cover 95% and there are 40,000 levels, you don't have a high-cardinality problem—you have a rare-category problem, and a frequency floor fixes it this afternoon.

**Grep for `groupby(...).mean()` near your target column (15 min).** Every hit is a candidate target-encoding leak. For each one, answer a single question: was this computed inside a cross-validation fold, or once over the whole dataset? The second answer means your reported metric is wrong right now.

**Diff your training vocabulary against last week's live traffic (30 min).** Pull the distinct values your model actually received in production over seven days and set-subtract the vocabulary the encoder was fit on. Anything in the difference has been silently zeroed or globally-meaned on every request since it appeared.

---

## Your Homework

### Exercise 1: Reproduce the Ad-Tech Leak (Time: ~45 minutes)

Take any dataset with a high-cardinality categorical column and a binary target. Target-encode it the naive way—group by, take the mean, map it back over the full training set—and record the cross-validated AUC. Then do it out-of-fold with smoothing and record it again. The first number will be higher. Write both numbers down and keep them, because the next time somebody shows you a model that jumped six points from one feature change, you will know precisely which question to ask.

### Exercise 2: Price Your Cardinality (Time: ~30 minutes)

For your three highest-cardinality categorical features, compute: total distinct levels, levels needed to cover 95% of rows, and the count of levels appearing fewer than five times. Then one-hot encode the raw column and record the matrix size in memory. You now have the actual cost of the default choice, in megabytes, which is a far more persuasive artifact in a design review than an opinion about encoders.

### Exercise 3: Train the Unknown (Time: ~1 hour)

Take a model you have already shipped. Retrain it with a preprocessing step that reassigns 5% of the rows in rare categories to an explicit `__UNKNOWN__` level. Compare offline metrics—they will barely move. Then send both versions a request containing a genuinely novel category and compare the predictions. The retrained model has a learned answer. The original one is guessing, and it has been guessing on every unfamiliar record since the day you deployed it.

---

## Bridge to Part V

Parts of this book so far have been about repair. Missing values, outliers, scales, categories: four chapters of taking the data you were given and making it fit to feed a model without lying to it. That work is necessary and it is not where models are won.

Part V is about the other half—not fixing what you have, but creating what you don't. The features that decide whether a model is useful usually don't exist in any source system. Nobody logs "days since this customer's last complaint," or "ratio of this transaction to this cardholder's rolling median," or "how many admins this account has lost this quarter." Somebody has to know the domain well enough to realize the number matters, and then build it.

Chapter 14 is about that construction: where good features actually come from, why the best ones are usually obvious in hindsight and invisible in advance, and why the automated feature-engineering tool that generates three thousand candidates for you will reliably miss the one that matters.

---

*P.S. — My favorite detail in the ad-tech story is that the model was right. Given a feature column containing the answer, learning to read the answer is correct behavior, efficient behavior, exactly what you paid for. The model held up its end of the deal perfectly. It just turned out that the deal, as written, was for a model that could predict the past.*
