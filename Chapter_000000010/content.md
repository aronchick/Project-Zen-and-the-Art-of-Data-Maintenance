# Chapter 10: Handling Missing Data and Imputation

## Or: The Hole Is the Message

A hospital system with eleven facilities and a bit under 400,000 admissions a year built a thirty-day readmission risk model. The point of it was a care-management program: twelve nurses who called discharged patients, walked through their medications, and made sure someone had actually booked the follow-up appointment. Nurses are expensive and there were twelve of them. So, like so many things nowadays, they decided to use a model to decide who got the call.

With a metric ton of data, they built it. Everything—EHR. Demographics, diagnosis codes, length of stay, prior admission history, and about sixty lab values—all went in, and it all validated respectably. Area under the curve (AUC) was 0.72, which for readmission is a perfectly credible number (readmission is really hard to predict). Then they shipped it, and nurses started working the list every morning.

Eight months later, readmissions had not moved... at all. The program came up for renewal and the CFO asked a reasonable question about what $1.4 million of nursing salary had purchased.

The answer was one line in the preprocessing script:

```python
df = df.fillna(df.median())
```

For those who don't know Python, that line finds every empty cell in the table and writes the column's median value into it. Just that ENTRY was empty, mind you—the patient had everything else. And in most cases, patching a hole with a typical value is the exact right thing to do! In a hospital, it is close to the worst thing you can do.

In this case, roughly 40% of the lab values were missing because nobody ordered the test. Which is not at all uncommon because in a hospital, nobody orders a test at random. A serum albumin gets drawn when a clinician is worried about nutrition. A lactate gets drawn when someone is worried about sepsis. The *existence* of that test is a physician's judgment about a patient, written into the record, for free.

So the null in that column was not an absence of information. It was a doctor saying "I looked at this person and I wasn't concerned." That is one of the most informative things in the entire chart, and `fillna(median)` painted over it with the lab value of an average sick person.

As a result, it broke in both directions at once. Patients with no labs drawn (the well ones) were assigned median-sick values, which pushed their risk scores upward. Patients who *did* have labs drawn had real values, but those real values now sat in a distribution containing an enormous artificial spike right at the center, which flattened the model's ability to tell them apart. All this means that the model confidently ranked a population it had been lied to about, and twelve nurses spent their days calling the wrong people.

They dropped the median fill, let the gradient-boosted model handle nulls natively, and added one binary column per lab: was this test ordered, yes or no. AUC went from 0.72 to 0.79. The strongest single predictor in the new model was not any lab *value*; it was whether a lactate had been drawn at all.

The missing data was the most predictive signal in the dataset, and yet the preprocessing step deleted it.

---

Missing data is data. I know that sounds counterintuitive, but it's true.

Every null in your table got there through a process. That process has a shape, and the shape is sometimes the answer to the exact question you're asking. Sometimes it's boring—a sensor dropping a packet, a form field that was optional, a join that doesn't match. Sometimes it's the whole story. Much of the discipline comes down to working out which one you're holding *before* you fill anything in.

Which brings us to a word worth nailing down. Imputation is the act of replacing a missing value with an inferred one. It does not recover the fact that was lost; it inserts a plausible substitute based on assumptions about why the value is missing.

Every imputation method in existence is a bet about the mechanism that created the gap. Bet wrong and you don't get an exception, a warning, or a failed test. You get a model that validates beautifully and is useless in a way nobody notices, which is the most expensive failure mode in this book because it takes eight months and a CFO to detect.

---

## 10.1 Understanding Missingness Mechanisms: MCAR, MAR, MNAR

The taxonomy comes from Donald Rubin in 1976 and it has survived fifty years which is crazy since that's like millennia in tech. Three mechanisms, and the practical consequences are wildly different.

**MCAR—Missing Completely At Random.** The probability that a value is missing has nothing to do with anything: not the missing value, not the other columns, not the time of day. This could be truly anything: a network switch dropped packets; a lab machine was down for maintenance on a Tuesday; anything. This is the only mechanism where deleting incomplete rows is statistically safe, because the rows you delete are a random sample of the rows you keep. You lose statistical power and you keep your correctness. The catch is that almost nothing in a real system is MCAR, and the people who assume it are usually assuming it because they didn't check.

**MAR—Missing At Random.** Terrible name, important idea. It means the missingness depends on data you *do* have. Income is missing more often for younger respondents—but you have age, so conditional on age, the missingness is random. This is the sweet spot: the information needed to correct the gap is sitting there in your other columns, which is exactly what model-based imputation exploits. Most serious imputation methods assume MAR.

**MNAR—Missing Not At Random.** The missingness depends on the value you can't see. High earners decline to state income. Patients who feel fine don't get labs drawn. A vibration sensor stops reporting *because* the machine shook itself apart. This is the most dangerous one because **you cannot detect MNAR from the data.** Confirming that missingness depends on the missing value would require the missing values.

The only way to get at MNAR is with the process that produced your dataset, and the only reliable way to learn it is Chapter 5's discipline: go find the human who built the collection path and ask them why a field would be empty. Thirty minutes with the clinical informatics lead would have saved that hospital eight months. Nobody had the conversation, because the data was *in the warehouse* and things in the warehouse feel like facts.

If you have the time (and if I've convinced you of anything, it should be that you really should have the time), go build a model that predicts your own missingness. Take the column you're worried about, create a binary target—`is_missing`—and throw every other feature at a quick gradient-boosted classifier. If it predicts at AUC 0.52, your missingness is unrelated to everything you can observe, and MCAR is at least defensible. If it predicts at 0.85, missingness is systematically tied to your observed data, and more usefully, the feature importances tell you *exactly what it's tied to*. In the hospital's case that model would have hit the ceiling in about a minute, with "admitted through the ED" and "prior ICU stay" at the top of the list, and somebody would have asked the right question a year earlier.

That diagnostic cannot distinguish MAR from MNAR—nothing can. But it will tell you loudly when you are not in MCAR, which is the assumption most pipelines are silently making.

---

## 10.2 Simple to Advanced Imputation Strategies

The biggest gain on the whole "getting your empty stuff into a better shape" ladder is the step from rung two to rung four. Going from mean imputation to indicators-plus-native-handling will buy you more than going from KNN to MICE ever will. People skip the cheap rungs because they feel unsophisticated and then spend three weeks tuning an iterative imputer.

The mechanisms for dealing with empties are BASICALLY one of the following:

1. **Deletion.** Drop rows with any missing value. Honest under MCAR, and it has a cost most people never compute. Five columns, each independently 10% missing, means a complete row survives with probability 0.9⁵ = 0.59. You just threw away 41% of your data to fix a 10% problem. Run that arithmetic on your own table before you reach for `dropna()`; it's usually worse than anyone guesses. And when the data isn't MCAR, deletion doesn't just cost you power—it silently reshapes your population. Drop every row missing income and you have built a model for people who answer questions about their income. Unless you have a death wish, I would not do this.

2. **Mean, median, mode.** The default, and the disaster from the cold open. Three things happen and all of them are bad. It shrinks variance, because you've added a pile of values with zero deviation from the center. It manufactures a spike in the distribution that was never in the world. And it attenuates every correlation that column participated in toward zero, because the imputed rows carry no relationship to anything. The deepest problem is one of *confidence*: after `fillna`, a fabricated value and a measured value are indistinguishable to the model, and the model will treat both as fact. This KIND OF works, but when it doesn't work, it can fail in pretty hidden ways. Almost always avoid.

3. **Constant and sentinel values.** Fill with -1, or 0, or `"UNKNOWN"`. This gets sneered at and it shouldn't be, because for tree-based models it's frequently the right answer. A tree can put a split at -0.5 and carve the missing population out into its own branch, which means the sentinel *preserves* the missingness signal instead of erasing it. For anything linear, distance-based, or gradient-descent-trained, it's a catastrophe: you've told a regression that these customers have negative tenure, and it will dutifully fit a coefficient to that. You can deal with certain cases with better filtering, but it's better than one and two.

4. **The missingness indicator.** Add a binary companion column—`albumin_was_measured`—alongside whatever you do with the value itself. This is the cheapest feature engineering that exists and it's the single highest-leverage move in this chapter. It converts an MNAR trap into an ordinary feature. In scikit-learn it's `SimpleImputer(add_indicator=True)` or the standalone `MissingIndicator`. The cost is column count, so use judgment: add it for columns where missingness plausibly means something, not reflexively for all 400 of them.

5. **KNN imputation.** Find the *k* most similar complete rows and borrow their values. Preserves local structure much better than a global mean. Three caveats and you will hit all three. It needs scaled inputs or the distance metric is dominated by whichever column happens to be measured in dollars (Chapter 12). It's expensive—you're doing a neighbor search per missing cell. And it will leak like a sieve if you fit it on the full dataset before splitting, which brings us to a rule we'll keep hitting: *the imputer is part of the model.*

6. **Iterative imputation and MICE.** Model each incomplete column as a function of all the others, cycle through the columns repeatedly until the estimates stop moving. This is `IterativeImputer` in scikit-learn and `mice` in R, and it handles MAR properly because it uses precisely the observed information that MAR says is sufficient. It works, with one large asterisk: the *M* in MICE stands for Multiple. The statistically correct procedure is to generate *m* different completed datasets, train *m* models, and pool the results, so that the uncertainty in your imputation propagates into your final estimate. Approximately nobody does this. They generate one dataset, train one model, and report a confidence interval that's too narrow because it pretends the imputed values were measured. If you're doing inference rather than prediction, do the multiple part. If you're doing prediction, understand that you've bought a point estimate and stop treating it as truth.

7. **Let the model handle it.** XGBoost, LightGBM, and CatBoost all learn a *default direction* at every split: when a value is missing, send it left or right, whichever reduces loss more. That's not a workaround, it's a learned imputation policy that's fit jointly with everything else and optimized for the actual objective. For a large share of tabular problems, the correct answer to "which imputation method?" is "none, plus an indicator, and use a gradient-boosted tree." Not sophisticated. Frequently best.

---

## 10.3 Deep Learning Approaches to Missing Data

Neural imputation is a real research area with real results, and it's also where a lot of teams go to feel advanced while making their lives worse. Both things are true.

**Denoising autoencoders** are the workhorse idea. Take your complete records, artificially corrupt them by masking random entries, and train a network to reconstruct the original. You've now got a model that knows the joint structure of your data well enough to fill gaps from context. Point it at your missing values and let it reconstruct.

**GAIN**—Generative Adversarial Imputation Nets, out of van der Schaar's group at ICML 2018—makes it adversarial. A generator fills in the missing entries and a discriminator tries to identify which entries were imputed versus observed. The generator wins by producing fills that are indistinguishable from real values, which is a much sharper training signal than reconstruction error alone. **Variational autoencoders** do a related job with an explicit probabilistic story, which has the nice property of giving you a *distribution* over plausible fills rather than one number. And for sequences, attention-based architectures handle irregular sampling and long gaps in a way that classical interpolation simply can't.

These methods, however advanced, are only really applicable for a few scenarios. High missingness rates, strong nonlinear structure among your features, enough data to train a second model, and a downstream task where imputation quality changes the answer. Outside that, which is most tabular business problems, they lose to a gradient-boosted tree with an indicator column, and they cost real money to keep alive.

Because, ultimately, a neural imputer *is a model*, which means it has to be trained, versioned, and deployed (you ARE doing that with all your models, right?). It drifts, needs monitoring, needs to run in the serving path within your latency budget. It also needs an entry in your feature store and a point-in-time-correct training history (Chapter 9), because if you fit the imputer on data from after the prediction timestamp, you've got leakage. If you're not careful, you won't be adding just a preprocessing step; you'll be adding a second production ML system whose failures are invisible because its outputs look like data.

Where they clearly win, however, they really win. Image and sensor data with structured gaps, where the spatial or temporal correlation is strong and a mean is absurd, signal reconstruction, and any case where you need *plausible complete records* for something other than a model. This could be a simulation, sharing a dataset that can't have holes in it, or generating synthetic data for a partner. When the imputed values are the deliverable rather than an input, spending a neural network on them makes sense.

---

## 10.4 Domain-Specific Imputation Techniques

The BEST imputation decisions are domain decisions, and they mostly come from knowing what the number physically *is*.

**Time series.** Forward-fill—carry the last observation forward—is the default, and whether it's right depends on if your quantity is a *state* or an *event*. A thermostat setpoint is a state: if nobody changed it, last week's value is still true, and forward-filling is not an approximation, it's correct. Transactions per minute is an event rate: forward-filling it invents traffic that never happened. Linear or spline interpolation suits smooth physical quantities—temperature, pressure, position. Anything with strong periodicity deserves seasonal decomposition first, so you impute the residual rather than the trend.

And one hard rule: **never backward-fill in a training set.** Backward-fill uses the future to explain the past. It is leakage, it will make your backtest gorgeous and your production system worthless. Forward-fill uses only the past. One of those is a method and one of those is a crime.

**Clinical and EHR data.** Missingness encodes clinical judgment, for example, what the medical professional decided to order (or not). Standard practice is last-value-carried-forward *within an encounter* with an explicit staleness cap—a creatinine from six hours ago is informative, a creatinine from nine days ago is a different patient.

**Financial data.** A missing price is not zero. A thinly traded asset that didn't trade today has a *stale* price, not a nonexistent one, and filling the return with 0 fabricates a day on which the market was flat. Filling in blanks will make any volatility model you build understate risk. Watch also for the rows that vanished entirely—survivorship bias is missing data with the evidence removed, and it's the reason so many backtested strategies only work on the funds that survived.

**Sensor and IoT data.** First, separate "no reading" from "a reading of zero"—a flow meter reporting 0 and a flow meter reporting nothing are opposite facts. Second, be paranoid: a sensor that stops reporting is very often a sensor that failed, and sensors fail under exactly the extreme conditions you built the model to detect. That's MNAR. The gap in your data is where the interesting thing happened.

**Survey and self-reported data.** The textbook MNAR case because it's so reliable. Nonresponse on income, weight, and anything embarrassing correlates with the value itself. Treat any imputation here as an assumption you are making about people who declined to answer, and write the assumption down somewhere a reviewer can find it.

**Geospatial data.** Spatial interpolation—kriging and friends—uses the fact that nearby locations are correlated, which is more information than any generic imputer has access to. Use it when the geography is real, and don't use it across boundaries that matter, because "nearby" and "similar" can get pretty hairy across made up things like borders.

---

## 10.5 Validating Imputation Quality

To state it plainly: **you cannot validate an imputation against ground truth, because if you had the ground truth the value wouldn't be missing.** Every validation strategy is therefore indirect, and you should use different ones depending on the question you have.

**How does your imputation method affect the data set entries?** Take your complete rows, knock values out under a mechanism you control, impute, and measure the error. This tells you how a method behaves under an assumed mechanism—and you should assume more than one. Knock values out completely at random, then knock them out conditioned on an observed column, then knock out the highest values specifically. Watch how each method degrades. Mean imputation will look almost respectable under MCAR and fall apart under the third, which is precisely the lesson worth internalizing.

**What does my imputation do to aggregate statistics, like distributions?** Overlay the imputed values against the observed ones. Mean imputation will often be incredibly obvious—a single spike where a distribution should be. Then check the correlation matrix before and after imputation. If your correlations moved substantially toward zero, your imputer diluted the structure you were trying to model.

**What changes downstream once I start imputing?** Does the model ACTUALLY get better? Cross-validated, with the imputer fit *inside* each fold. Which brings us to the most common anti-pattern in this chapter's territory.

Fitting the imputer on the entire dataset before splitting for training / test is leakage, because the median you filled the training rows with was computed using the test rows. It is a small leak, it inflates your validation score by a modest and entirely fake amount, and it is present in an alarming share of the notebooks in the world. The point people miss is that an imputer *has fitted parameters* of its own—that median is something it learned from data. Anything with fitted parameters belongs inside the `Pipeline` object and inside the fold, alongside the scaler and the encoder.

**Does imputation affect the sensitivity analysis?** Impute three defensible ways, retrain, and see whether your conclusion survives. If your feature ranking reorders or your business recommendation flips depending on how you filled the nulls, you have to go back. You have found an artifact of a preprocessing choice, and the appropriate action is to go get better data, not to pick the version you like.

---

## 10.6 Production Considerations for Missing Data

Everything above concerns the offline world; production, as usual, adds failure modes that don't exist in a notebook.

**The imputer is a train/serve skew machine.** Your training pipeline imputed with the training median. What does your serving path use? If it recomputes a median from recent live traffic, you have a new problem. Two code paths, one feature name, values that diverge as the live distribution moves. Freeze imputation parameters as versioned artifacts at training time, and then ship them with the model. The median is not a computation, it's a constant you learned.

**The missingness rate is a monitoring signal.** A column that has been 3% null for two years and is suddenly 40% null means an upstream integration broke, a vendor changed a schema, or a field got deprecated by a team that didn't know you existed. That event is usually more urgent than any distributional drift, it's trivial to detect, and almost nobody alerts on it. Track null rate per column per day, and when (not if) it moves, something upstream changed, and you now know before your model degrades instead of after.

**Plan for nulls you never trained on.** You saw zero missing values in `customer_tenure` across two years of training data, so the question never came up. Production will hand you one on a Thursday. What happens—an exception, a silent zero, a NaN that propagates through to a prediction of `nan` that some downstream service interprets as 0.0? Decide deliberately now, or discover it during an incident.

**Budget for serving latency.** KNN and iterative imputers are expensive at inference. Teams that train with MICE and serve with a median because MICE couldn't fit the latency budget face train/serve skew by another name and just as fatal. If you can't run it in the serving path, you can't use it in the training path either.

**Build the escalation ladder.** When a value is missing at inference, there's a sequence of increasingly degraded options: use the real value, use a recent cached value for that entity, use the frozen training-time default, or refuse to score. This last choice is something that a lot of people should choose more often. A model that declines to make a prediction when its inputs are too degraded is often more valuable than one that confidently predicts from a fabricated input—especially when a human is on the other end deciding about a loan, a diagnosis, or a shutdown. Most teams never build the refuse path, because it requires admitting the model has limits. Build it anyway, and make sure something downstream knows how to handle the refusal.

---

## Quick Wins: Null Audits You Can Run Today

**Count nulls by column AND by time (15 min).** Everyone runs `df.isnull().sum()`. Almost nobody groups it by month, which is a real shame. A column that went from 2% to 30% missing in March is not a data-quality problem to be imputed away—it's an incident nobody logged, and the date will tell you which upstream change caused it.

**Predict your own missingness (20 min).** For your three most-missing columns, build a quick classifier with `is_missing` as the target and everything else as features. If AUC is near 0.5, relax. If it's 0.8 or higher, read the feature importances and you have just learned the mechanism you were about to assume away.

**Add indicators and retrain (30 min).** Take your top five columns by null rate, add a binary was-it-present flag for each, retrain, compare. This is thirty minutes and one line of code, and it either does nothing—fine, now you know—or it moves your metric enough to reframe the project.

**Find where your imputer is fit (10 min).** Search your training code for `fillna`, `SimpleImputer`, and `median()`. For each hit, answer one question: does this run before or after the train/test split? Every one that runs before is inflating your validation score right now.

---

## Your Homework

### Exercise 1: The Mechanism Interview (Time: ~45 minutes, mostly talking)

Pick the column with the most nulls in your most important dataset. Find the person or team responsible for the system that produces that field and ask them one question: "under what circumstances is this empty?" Write down their answer verbatim. Then classify it—MCAR, MAR, or MNAR—and compare that to whatever your pipeline currently assumes. Where those two disagree, the pipeline is wrong and the data is right. This exercise fails for most people not because the answer is hard but because they can't find who to ask, and *that's* the finding.

### Exercise 2: Break Your Own Data (Time: ~1 hour)

Take a complete dataset. Create three corrupted copies: one where you delete 20% of a column at random, one where you delete it conditioned on another column's value, and one where you delete the top 20% of values specifically. Impute all three with mean, with an indicator plus native tree handling, and with `IterativeImputer`. Train the same model on each of the nine combinations and record the test performance. You will produce a small table that tells you which methods are robust to which mechanisms, and you will never again reach for `fillna(mean)` without a small flinch.

### Exercise 3: The Serving Null Test (Time: ~30 minutes, do this in staging)

Send your production model a request with a null in a field that has never been null in training. Then one with a null in your most important feature. Then one with every optional field empty. Record what comes back for each: an error, a prediction, or a prediction that's silently garbage. Write down what *should* happen in each case. If the two lists don't match, you've found the incident you were going to have anyway, and you've found it on a Tuesday afternoon instead of at 3 a.m.

---

## Bridge to Chapter 11

Missing values are the gaps in your data. The next chapter is about the opposite problem: the values that are *there*, that are real, and that are so far from everything else that somebody is going to want to delete them.

The instinct is the same instinct, and it's wrong in the same way. Just as a null is not merely an absence to be filled, an extreme value is not merely an error to be removed. Sometimes it's a fat-fingered decimal point. Sometimes it's the most important observation in the dataset—the machine that was about to fail, the transaction that was actually fraud, the claim that will consume a third of your annual loss budget. The `fillna` reflex and the three-sigma reflex are the same reflex: reaching for a default that makes the data look better without asking what the data was trying to say.

Chapter 11 is about outliers—how to find them, how to tell the errors from the signals, and what it costs when you clean away the exact thing you were paid to predict.

---

*P.S. — My favorite thing about nulls is that everyone treats them as a nuisance to be disposed of before the real work starts, when the null rate is very often the highest-signal column in the entire table. It is the one field that records what someone chose not to do, and people are far more honest in what they skip than in what they fill out. Somewhere in your warehouse there is a column that is 60% empty, and that emptiness is a better predictor of your target than the eleven features your last sprint was spent engineering. Nobody will ever put it in a slide deck, because "we added a boolean" is a terrible slide.*
