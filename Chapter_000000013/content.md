# Chapter 13: Encoding Strategies for Categorical Variables

## Or: The Model Memorized the Answer Key

A demand-side platform bid on display ad impressions in real time. Roughly $180 million a year of media spend flowed through it, and the platform took a cut, so the whole business rested on one prediction made in under ten milliseconds: given this impression, how likely is a click? Bid accordingly.

The most predictive thing about an impression is where it appears. Not the site—the *placement*: a specific ad slot, on a specific page template, on a specific property. The training data had 2.3 million distinct `placement_id` values.

One-hot encoding 2.3 million categories—one yes/no column per category—was obviously off the table, so an engineer reached for target encoding. Replace each placement with the average click-through rate (CTR) observed for that placement. One column instead of 2.3 million. Directly, obviously informative.

```python
means = train.groupby("placement_id")["clicked"].mean()
train["placement_ctr"] = train["placement_id"].map(means)
```

Offline AUC went from 0.71 to 0.91 on the same model, cross-validated over the same already-encoded training frame. (AUC—area under the ROC curve—is the standard scorecard for a yes/no predictor: the probability that the model ranks a randomly chosen clicked impression above a randomly chosen unclicked one. 0.5 is a coin flip, 1.0 is perfect, and `roc_auc_score(y_true, y_score)` computes it. Cross-validation, or CV, scores the model on slices of the training data it was not fit on, and is supposed to be the thing that catches what happened next.) That is not an improvement anyone questions on a Friday. They shipped it.

Here is what those two lines actually built. Sixty percent of those 2.3 million placements appeared fewer than five times in the training window. For a placement that appeared exactly once, its "average CTR" is that single row's label. Not correlated with the label. *Is* the label, written into the feature column, sitting right next to it at training time.

The model did what any competent model does when you hand it the answer key: it learned to read the answer key. `placement_ctr == 1.0` means clicked. `placement_ctr == 0.0` means not clicked. It scored 0.91 because it was being graded on a test it had already been given.

In production, none of that transfers. A live impression on a rare placement gets whatever mean was computed from its handful of historical rows, which is a number like 1.0 or 0.5 derived from two observations, and the model treats that number with the same total confidence it learned in training. So the bidder paid up—hard—for inventory it was certain about and wrong about. Live AUC was 0.62. Against a 0.71 baseline the model had replaced.

Eleven weeks and $8.7 million in misdirected media spend later, an analyst plotted predicted CTR against realized CTR bucketed by how many times each placement had been seen before, and the whole thing fell out of the chart in about four minutes.

The fix took a day. Compute the encoding out-of-fold so a row never contributes to its own feature. Smooth every category's average toward the global average in proportion to how little evidence supports it. Bucket everything below a frequency floor into an explicit rare category. Offline AUC dropped to 0.78. Live AUC came in at 0.77.

The number on the slide went down. The business went up. That gap—0.91 offline versus 0.78 offline—was never performance. It was the size of the lie.

---

A categorical variable is a column whose values are labels drawn from a set—country, merchant, device type, diagnosis code—rather than quantities you can add or average. Put more bluntly, it is a fact about the world that does not support arithmetic. There is no mean of a country, no square root of a diagnosis code, no sensible answer to "what is `merchant_id` 4471 minus `merchant_id` 3902." The number of distinct labels a column can take is its **cardinality**: a device-type column with five values is low-cardinality, and a placement column with 2.3 million is about as high as cardinality gets.

Every model you will ever train, on the other hand, runs on arithmetic. **Encoding** is the step in between: turning each label into one or more numbers the model can compute with. Since the labels have no arithmetic of their own, encoding is really the act of *inventing* arithmetic for something that doesn't have any, and every encoding scheme is a claim about the structure of the thing you're encoding. One-hot (one yes/no column per category) claims the categories are mutually exclusive and unrelated. Ordinal (one integer per category, in sequence) claims they have an order. Target encoding (replace each category with the average outcome seen for it) claims a category's history predicts its future. Embeddings (a short learned vector per category) claim there is latent structure worth learning.

Get the claim right and the model gets a real head start. Get it wrong and you have done what Chapter 2 warned about, one layer deeper: you've told the model something false about the world, in a form it has no way to question.

---

## 13.1 Understanding Categorical Types: Nominal, Ordinal, and Cyclical

Before you pick an encoder, you have to know which of three things you're holding. This is not a property of the column. It's a claim about the world that the column describes, and the column will not tell you which one is true.

**Nominal** categories have no order at all. Country. Merchant. Device type. Diagnosis code. Product category. The only relationship between two nominal values is same-or-different. Any encoding that implies more than that is inventing a fact.

**Ordinal** categories have a real order but no guaranteed spacing. Small, medium, large. Credit ratings from AAA down to D. Education level. A five-point pain scale. The order is genuine—large really is more than medium—but the gaps are not equal and are usually unknowable. When you map small/medium/large to 1/2/3, you have not just encoded the order. You have asserted that the distance from small to medium equals the distance from medium to large, and that going from small to large is exactly twice the step from small to medium. Sometimes that's harmless. For garment sizes it's roughly true. For credit ratings it is spectacularly false: the distance from AAA to AA is nothing like the distance from BBB to BB, which is the exact boundary where institutional mandates force selling.

If you know the real spacing, encode the real spacing. Map credit ratings to historical default rates. Map T-shirt sizes to chest measurements. You are allowed to put domain knowledge into the numbers; that's the whole point of Chapter 14.

**Cyclical** categories have an order that wraps. Month, day of week, hour, compass bearing, wind direction, angle. December is month 12 and January is month 1, and every linear encoding you can write says they are eleven units apart when they are in fact adjacent. Same for hour 23 and hour 0, which are sixty minutes apart.

The standard fix is to project onto a circle:

```python
df["hour_sin"] = np.sin(2 * np.pi * df["hour"] / 24)
df["hour_cos"] = np.cos(2 * np.pi * df["hour"] / 24)
```

You need both. Sine alone maps 3 a.m. and 9 a.m. to exactly the same value, which just moves the collision somewhere less obvious. The pair together gives every hour a unique position on a circle where hour 23 and hour 0 sit next to each other, which is where they belong.

One caveat worth internalizing: tree-based models handle cyclical variables tolerably well without this, because a tree can carve out `hour >= 22 OR hour <= 5` given enough depth. Linear models and distance-based models cannot, ever. If you're running k-NN (k-nearest-neighbors, which predicts from whichever training rows sit closest by distance) or a linear model on raw hour-of-day, your model believes midnight and 11 p.m. are maximally different, and it will keep believing that no matter how much data you give it.

---

## 13.2 Basic to Advanced Encoding Techniques

**One-hot encoding** is the correct default and you should feel no shame about it. One binary column per category, exactly one of them hot per row. It makes the minimum possible claim—these things are distinct—and it works with everything. In scikit-learn it is `OneHotEncoder()`, fit on the training frame and applied to everything after. `pd.get_dummies(df["device"])` produces the same columns for a quick look, but it has no fitted vocabulary—the columns it emits depend on whatever values happen to be in the frame you hand it—so a batch that is missing one category comes back one column narrower, and anything you intend to serve should use the estimator.

Its cost is width. Fifteen categories is fifteen columns and nobody notices. Fifteen thousand categories is fifteen thousand mostly-zero columns, and now you have a sparse matrix, a memory problem, and a model that has to estimate fifteen thousand coefficients from data that gives it three examples of most of them. A sparse matrix stores only the nonzero entries and where they sit, which is the only sane way to hold fifteen thousand mostly-zero columns; `OneHotEncoder` returns one by default, and `sparse_output=False` hands you the ordinary dense array, every zero included, when a downstream library insists.

The drop-one-level question—`OneHotEncoder(drop="first")`, or `drop_first=True` in `pd.get_dummies`—comes up constantly and the answer depends entirely on the model. For unregularized linear models (no penalty on how large the coefficients get), keeping all levels plus an intercept gives you perfect multicollinearity, better known as the dummy variable trap: the one-hot columns always sum to exactly one, which is the intercept column, so one of them is a perfect copy of information already in the others, there is no single best set of coefficients, and the fit becomes unstable. Drop one, and it becomes the reference level everything else is measured against. For regularized linear models it matters less but changes what the penalty means—dropped levels get their effect absorbed into the intercept and escape shrinkage. For trees, dropping a level is mildly harmful: it makes one category expressible only as the absence of all the others, which costs the tree extra depth to say.

**Ordinal encoding**—map each category to an integer—is right when the thing is truly ordinal and you've thought about spacing, and wrong the rest of the time. Always hand the encoder the order yourself: `OrdinalEncoder(categories=[["small", "medium", "large"]])`. Left to its defaults, `OrdinalEncoder()` sorts the labels alphabetically, which puts large below medium below small and quietly encodes an order nobody meant. There is a common defense that it's fine for tree models because a tree can recover any grouping through repeated splits. This is true and still not free: with an arbitrary integer assignment, expressing "categories 3, 7, and 12 behave alike" costs several splits and several levels of depth, and every split spent on reconstruction is a split not spent on signal.

**Frequency and count encoding** replaces each category with how often it appears. In pandas that is `df["merchant_id"].map(df["merchant_id"].value_counts())`, with the counts taken from the training frame and reused as-is at serving time. Cheap, one column, no target involvement so none of the label-leakage risk that 13.3 is about, and often more predictive than it has any right to be—for merchants, sellers, and users, "how much do we see this thing" carries real information about size, tenure, and legitimacy. The obvious failure is collision: two categories with identical counts become identical to the model. Usually acceptable, occasionally not.

**Binary and BaseN encoding** compress cardinality into log-scaled columns by writing the category index in base 2 or base N. Both live in `category_encoders`, a third-party library of encoders that scikit-learn doesn't ship (`import category_encoders as ce`): `ce.BinaryEncoder(cols=["merchant_id"])` and `ce.BaseNEncoder(cols=["merchant_id"], base=4)`. They were a reasonable middle ground when memory was tight. The bit patterns are arbitrary, so the model has to learn to reassemble a meaning from digits that have none, and in practice hashing or embeddings do the same job better. Know it exists; reach for it rarely.

**Native categorical handling** is what you should actually be doing on tree models in 2026. LightGBM and XGBoost (with `enable_categorical=True`) accept categorical columns directly and split on them properly—partitioning the category set rather than testing one value at a time. CatBoost accepts them too, but gets there another way, converting each category to an ordered target statistic and then splitting on that number. This is strictly better than one-hot for trees at moderate-to-high cardinality, it's one parameter, and a startling number of teams don't know it's there.

```python
import lightgbm as lgb

for c in cat_cols:
    X_train[c] = X_train[c].astype("category")

model = lgb.LGBMClassifier()
model.fit(X_train, y_train, categorical_feature=cat_cols)
```

That is very nearly the whole intervention. The one catch is that the category dtype carries a code mapping. LightGBM's Python wrapper remembers the training categories and re-applies them to pandas input at predict time, but the moment you hand it integer codes you computed yourself, or use the same trick with a library that doesn't remember, serving data has to present the same categories in the same order or the codes shift underneath you—13.5's problem, arriving early. Before you build anything more sophisticated, try this and write down the number.

---

## 13.3 Target-Based Encoding and Regularization

Target encoding—also sold under the name mean encoding; this book uses target encoding throughout—replaces a category with a statistic of the target computed over that category: mean CTR per placement, default rate per ZIP, average basket size per store. It is genuinely powerful. It collapses any cardinality to a single dense column, and that column is directly aimed at the thing you're predicting.

It is also the most reliable way there is to leak your labels into your features, and the leak is invisible in every offline metric you have. That is *target leakage*: information about the label—the thing you are trying to predict—finding its way into the features the model trains on. The model then learns to read the answer instead of predicting it, every offline score inflates, and nothing in production inflates with it.

You already saw the mechanism. Compute a category's mean over the full training set, and every row in that category has contributed its own label to its own feature value. For a category with ten thousand rows, that contribution is one part in ten thousand and nobody cares. For a category with three rows, the feature is a third of the answer. And in every real high-cardinality dataset, the distribution is long-tailed—a handful of categories account for most of the rows and a vast number appear only a few times each—so most of your categories live in the regime where the leak dominates.

Two defenses, and you want both.

**Out-of-fold computation.** Split the training data into folds. For rows in fold *k*, compute the encoding using only the other folds. A row's own label is then structurally incapable of reaching its own feature. Use time-ordered folds or an expanding window if the data has a time axis: with random folds, a row is still encoded from labels that, within its own category, happened after it. This is the same discipline as Chapter 9's point-in-time correctness, wearing different clothes: never let a row see information it wouldn't have had. By hand, it is a loop over folds:

```python
from sklearn.model_selection import KFold

oof = pd.Series(np.nan, index=train.index)
for fit_idx, enc_idx in KFold(5, shuffle=True, random_state=0).split(train):
    means = train.iloc[fit_idx].groupby("placement_id")["clicked"].mean()
    oof.iloc[enc_idx] = train["placement_id"].iloc[enc_idx].map(means).to_numpy()
train["placement_ctr"] = oof.fillna(train["clicked"].mean())
```

Swap `KFold` for `TimeSeriesSplit` when the data has a time axis. The `fillna` catches placements that appear in only one fold and so have no history in the other four; they get the global mean, which is the same fallback 13.5 comes back to.

**Smoothing toward the prior.** Even out-of-fold, a category with two observations produces a noisy estimate that the model has no way to discount. The *prior* is what you would believe about a category before seeing a single row of it—here, the global mean—and smoothing (or shrinkage; same idea, different textbook) pulls every category's own average toward that prior, weighted by how much evidence backs it:

```python
smoothed = (counts * category_mean + m * global_mean) / (counts + m)
```

Read `m` as "how many observations of evidence I demand before I start believing a category's own average." At `m = 20`, a category with 2 rows lands almost entirely on the global mean, a category with 20 rows sits halfway, and a category with 5,000 rows is essentially its own average. Tune `m`; it matters more than most of the hyperparameters—the knobs set before training rather than learned from it—that people do tune. Scikit-learn's `TargetEncoder` calls it `smooth`, and will estimate it for you if you leave it on `"auto"`.

CatBoost's ordered target statistics are the most rigorous version of this idea: it draws a random permutation of the data and encodes each row using only rows that came before it, which gets you out-of-fold behavior without a fold structure. That permutation is random unless you set `has_time=True`, so it respects your ordering only if you tell it to. If you're already using CatBoost, you're getting this for free and should not hand-roll a replacement. For any other model, `ce.CatBoostEncoder(cols=cat_cols)` applies the same ordered scheme as a standalone transformer; it walks the rows in the order you give it, so shuffle first—or sort by time, if time matters—before you fit.

Scikit-learn shipped a `TargetEncoder` in version 1.3 that does the cross-fitting internally—cross-fitting being scikit-learn's name for exactly the out-of-fold computation above—which removed most people's excuse for the naive version:

```python
from sklearn.preprocessing import TargetEncoder, StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score

pre = ColumnTransformer([
    ("cat", TargetEncoder(smooth=20.0), cat_cols),
    ("num", StandardScaler(), num_cols),
])
pipe = Pipeline([("pre", pre), ("model", LogisticRegression(max_iter=1000))])
cross_val_score(pipe, X, y, cv=5, scoring="roc_auc")
```

One detail worth internalizing, because it bites people who hand-roll this: the cross-fitting happens in `fit_transform`, not in `fit` followed by `transform`. Calling `fit(X, y).transform(X)` on your training data gives you the leaky version. A `Pipeline` calls `fit_transform` for you, which is one more reason to use one.

Use the `Pipeline`. It refits the encoder inside each CV fold rather than once over everything, which is the same leak Chapter 12 covered for scalers and which is far more dangerous here—a scaler leaks distribution shape, a target encoder leaks the answer.

One trap worth naming: **leave-one-out encoding** (`ce.LeaveOneOutEncoder(cols=cat_cols)`), which computes each row's encoding from every other row in its category, sounds like the strictest possible fix and is in fact worse than smoothed out-of-fold. It leaks in a subtler direction—the encoded value becomes systematically *anti*-correlated with the row's own label, which a model can learn to invert. If it seems too clever, it is.

---

## 13.4 High Cardinality Solutions: Hashing and Entity Embeddings

High cardinality shows up wherever your data contains identifiers rather than descriptions: merchant IDs, user IDs, SKUs, placements, ZIP codes, ICD-10 diagnosis codes, IP blocks. Hundreds of thousands to millions of levels, a long tail where most levels appear a handful of times, and new levels arriving every day.

**Rare-category bucketing** is the first thing to try and the most underused tool in this chapter. Take everything below a frequency floor and call it `__RARE__`. You typically lose nothing—a category seen twice supports no reliable estimate anyway—and you convert an unbounded vocabulary into a bounded one, which fixes memory, fixes the noisy-estimate problem, and gives you a natural home for unseen categories at serving time. Scikit-learn's `OneHotEncoder` has this built in: `OneHotEncoder(min_frequency=50, handle_unknown="infrequent_if_exist")` folds every category seen fewer than 50 times in training into one infrequent column and routes never-seen categories into it too, and `max_categories` caps the total width instead. Set them.

**The hashing trick** maps each category through a hash function—a deterministic recipe that turns any string into a number in a fixed range, with no lookup table—into a fixed number of buckets, and the bucket number becomes the column. Fixed memory regardless of cardinality, no vocabulary to store or ship, no training-time knowledge of the category set required, and unseen categories are handled automatically because a hash function has an opinion about every possible string.

```python
from sklearn.feature_extraction import FeatureHasher

hasher = FeatureHasher(n_features=2**18, input_type="string")
X_hashed = hasher.transform(df["merchant_id"].astype(str).apply(lambda s: [s]))
```

The cost is collisions—two unrelated merchants sharing a bucket—and total loss of interpretability, since you cannot invert a hash to ask which category a coefficient belongs to. Collisions bother people more than they should. A collision hurts most when both colliding categories are individually informative *and* point in opposite directions, and with buckets set to several times your effective cardinality that intersection is small. Size the space generously; powers of two, and more than you think.

**Entity embeddings** learn a dense vector per category—a short list of numbers, say sixteen of them, that stands in for the category and is adjusted during training until categories that behave alike end up with similar numbers—the way word embeddings work in language models. In PyTorch the whole mechanism is one layer, `nn.Embedding(num_embeddings=cardinality + 1, embedding_dim=16)`, with the `+ 1` reserving a slot for the unknown category that 13.5 is about; the model looks up each category's row and trains it along with everything else. The technique got its reputation from a Kaggle solution to Rossmann store sales forecasting, where embedding store IDs took third place against entries built on heavy hand-engineered features—competitive accuracy for a fraction of the feature work. A common starting rule for dimension is `min(50, (cardinality + 1) // 2)`, which nobody should treat as more than a starting point.

What you get beyond accuracy is a similarity space, and that is often the more valuable artifact. After training, stores that behave alike sit near each other in embedding space, whether or not anyone ever labeled them as similar. Merchandising teams will use that map long after the model that produced it is retired.

What you pay is real: you need a neural network or a library that supports embedding layers, you need enough data per category for the vectors to mean anything, and you have added a component that requires its own versioning and its own explanation to whoever audits your model. Embeddings are worth it when cardinality is high, data is plentiful, and you have reason to believe the categories have latent structure. They are not worth it because they sound better in a review.

---

## 13.5 Handling Unknown Categories in Production

The problem in this section ends more production models than anything else in this part of the book, and most treatments of encoding give it a paragraph.

Your encoder was fit on a fixed vocabulary. Production is not fixed. A new merchant onboards. A vendor adds a device type. Someone in operations creates a status code on a Tuesday afternoon to unblock a customer. Now a value arrives that the encoder has never seen, and one of three things happens.

**It raises.** `OneHotEncoder` with default settings throws on unknown input. Your inference service returns a 500. This is the *good* outcome, because you find out.

**It silently zeroes.** With `handle_unknown="ignore"`, an unknown value produces a row of all zeros. That is not a neutral outcome; it is the encoder asserting "this record belongs to none of the known categories," which for a linear model means the entire categorical contribution vanishes and the prediction shifts toward the intercept. Worse, if you also dropped a reference level, all-zeros already means that reference level, so the encoder is now asserting the unknown *is* the reference category. Every unknown value gets the same silent, confident, wrong answer. `handle_unknown="infrequent_if_exist"` is usually closer to what you wanted, because it routes unknowns into the infrequent bucket you already trained on, which at least has a learned response. Read that parameter name carefully, though: *if exist*. If you never set `min_frequency` or `max_categories`, there is no infrequent bucket, and the encoder falls back to the all-zeros behavior above without telling you. (It only warns if you also set `drop`, and the warning then claims the unknowns went to an infrequent category that doesn't exist. `handle_unknown="warn"` behaves like `infrequent_if_exist` and always warns, which is the setting to use when you want to find out.) Two settings, and one of them is silently the other.

**It gets the global mean.** Target encoders typically fall back to the prior for unseen categories—scikit-learn's `TargetEncoder` uses the training-set target mean, with no setting to change it—which is defensible and still needs to be a decision you made rather than a default you inherited.

(`OrdinalEncoder` offers the same menu under different names: it raises by default, and `handle_unknown="use_encoded_value", unknown_value=-1` assigns unknowns a sentinel code. The model has never seen a single training row with that code, so the sentinel is only useful if you also do what comes next.)

The real fix is to train the model on unknowns so it has a learned response to them. You cannot expect sensible behavior on a case you never showed it. The technique is to reserve an explicit `__UNKNOWN__` category and, during training, randomly reassign a small fraction of rare-category rows into it:

```python
counts = df["merchant_id"].map(df["merchant_id"].value_counts())
flip = (counts < 5) & (rng.random(len(df)) < 0.05)
df.loc[flip, "merchant_id"] = "__UNKNOWN__"
```

Three lines on the training frame, where `rng` is any `np.random.default_rng(seed)`. At serving time, map anything outside the training vocabulary to the same string before it reaches the encoder—`s.where(s.isin(vocab), "__UNKNOWN__")`—so the learned response actually fires. The model learns what to do when it doesn't recognize something, which is exactly the skill you need it to have on Tuesday.

Then monitor it. **Unknown rate per categorical feature belongs in your monitoring dashboard next to your accuracy metrics** (Chapter 25 covers the wider practice). A feature that runs at 0.3% unknowns for a year and jumps to 12% overnight is not a modeling event. It is an upstream schema change nobody told you about, and it is exactly what the data contracts in Chapter 5 exist to prevent.

Two variants of this failure are worth naming separately because monitoring the unknown rate will not catch either one.

**Category drift with a stable vocabulary.** The string `PENDING` was in your training data and is still arriving, but after a product launch it means something different than it used to. No unknown, no error, no alert. Your encoder maps it to the same coefficient it learned from the old meaning. Only a contract or a human catches this.

**Encoding skew between training and serving.** Training encoded `merchant_category` from the warehouse, where it is title case and trimmed. Serving reads the same conceptual field from an upstream service where it arrives as `"restaurant "`, lowercase with a trailing space. Different string, therefore unknown category, therefore silent zeros on a feature that mattered. This is Chapter 2's category explosion and Chapter 9's train/serve skew—the training path and the serving path computing the "same" feature two different ways—arriving together, through the encoder rather than the feature pipeline, and the defense is the same: one definition, computed once, shared by both paths.

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
| 15–1,000 levels, linear model | Target encoding, out-of-fold + smoothed | Fitting the encoder outside the CV fold |
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

**Count your tail (10 min).** For each categorical column, print what fraction of levels account for 95% of rows: `vc = df[c].value_counts(normalize=True)`, then `(vc.cumsum() < 0.95).sum() + 1` is the number of levels it takes. If ten levels cover 95% and there are 40,000 levels, you don't have a high-cardinality problem—you have a rare-category problem, and a frequency floor fixes it this afternoon.

**Grep for `groupby(...).mean()` near your target column (15 min).** Every hit is a candidate target-encoding leak. For each one, answer a single question: was this computed inside a cross-validation fold, or once over the whole dataset? The second answer means your reported metric is wrong right now.

**Diff your training vocabulary against last week's live traffic (30 min).** Pull the distinct values your model actually received in production over seven days and set-subtract the vocabulary the encoder was fit on: `set(live[c]) - set(enc.categories_[i])`, where `i` is the column's position in the encoder. Anything in the difference has been silently zeroed or globally-meaned on every request since it appeared.

---

## Your Homework

### Exercise 1: Reproduce the Ad-Tech Leak (Time: ~45 minutes)

Take any dataset with a high-cardinality categorical column and a binary target. Target-encode it the naive way—group by, take the mean, map it back over the full training set—and record the cross-validated AUC: `cross_val_score(model, X, y, cv=5, scoring="roc_auc")` with the encoded column already sitting in `X`. Then do it out-of-fold with smoothing—the `Pipeline` from 13.3, scored the same way—and record it again. The first number will be higher. Write both numbers down and keep them, because the next time somebody shows you a model that jumped six points from one feature change, you will know precisely which question to ask.

### Exercise 2: Price Your Cardinality (Time: ~30 minutes)

For your three highest-cardinality categorical features, compute: total distinct levels (`df[c].nunique()`), levels needed to cover 95% of rows (the Quick Win above), and the count of levels appearing fewer than five times (`(df[c].value_counts() < 5).sum()`). Then one-hot encode the raw column to a dense matrix—`OneHotEncoder(sparse_output=False).fit_transform(df[[c]])`—and record its size in memory from `.nbytes`. You now have the actual cost of the default choice, in megabytes, which is a far more persuasive artifact in a design review than an opinion about encoders.

### Exercise 3: Train the Unknown (Time: ~1 hour)

Take a model you have already shipped. Retrain it with a preprocessing step that reassigns 5% of the rows in rare categories to an explicit `__UNKNOWN__` level. Compare offline metrics—they will barely move. Then send both versions a request containing a genuinely novel category and compare the predictions. The retrained model has a learned answer. The original one is guessing, and it has been guessing on every unfamiliar record since the day you deployed it.

---

## Bridge to Part V

This part of the book has been about repair. Missing values, outliers, scales, categories: four chapters of taking the data you were given and making it fit to feed a model without lying to it. That work is necessary and it is not where models are won.

Part V is about the other half—not fixing what you have, but creating what you don't. The features that decide whether a model is useful usually don't exist in any source system. Nobody logs "days since this customer's last complaint," or "ratio of this transaction to this cardholder's rolling median," or "how many admins this account has lost this quarter." Somebody has to know the domain well enough to realize the number matters, and then build it.

Chapter 14 is about that construction: where good features actually come from, why the best ones are usually obvious in hindsight and invisible in advance, and why the automated feature-engineering tool that generates three thousand candidates for you will reliably miss the one that matters.

---

*P.S. — My favorite detail in the ad-tech story is that the model was right. Given a feature column containing the answer, learning to read the answer is correct behavior, efficient behavior, exactly what you paid for. The model held up its end of the deal perfectly. It just turned out that the deal, as written, was for a model that could predict the past.*
