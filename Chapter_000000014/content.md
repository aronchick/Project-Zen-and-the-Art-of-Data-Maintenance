# Chapter 14: The Art of Feature Creation

## Or: Three Thousand Features and One Good Idea

A B2B software company with about 11,000 accounts and $47 million in ARR wanted to know which customers were going to leave. Reasonable ask. They had nine people in customer success, a renewal calendar, and no way to decide who to call first.

The data team did the modern thing. They pointed an automated feature-engineering tool at the event warehouse—accounts, users, sessions, feature events, support tickets, invoices—turned the depth up, and let it run. It produced 3,400 features. Gradient boosting on top of that hit 0.83 AUC on a held-out set of past renewals.

Nobody had to argue for that model. It shipped.

Eight months on, the CS team had stopped opening the dashboard. Not because it was wrong. Because of *when* it was right.

Every one of those 3,400 features was an aggregate of product usage: logins per week, sessions per seat, distinct features touched per month, tickets per quarter. All perfectly real signals, and all of them measuring a thing that only moves once an account is already dying. The model lit up reliably at about thirty days from renewal, when weekly active users had fallen off a cliff.

Thirty days out, there is no save. The customer has already run a bake-off, already picked a replacement, already told their own boss. The only lever left is discounting, and discounting a customer who has mentally left just means losing them next year at a lower price. The CS team needed a hundred and twenty days to do anything real: an exec sponsor call, a business review, a rescue plan.

At 120 days out, the 3,400-feature model scored 0.58. Coin flip with extra steps.

What finally cracked it wasn't a model change. A CSM said the thing that everyone in the field already knew and nobody had written down: **accounts don't die when usage drops. Usage drops because the account already died.** The account dies when the champion leaves—the one person who fought for the purchase, ran the rollout, and defended the line item. When they take another job, the renewal is on a countdown that nothing in the product telemetry can see.

Nobody logs "the champion left." But two things nearby are logged. So they built exactly two features:

- days since the account's highest-permission user last logged in
- the trend in the count of distinct admin users over the trailing 90 days

Same model, same training data, those two columns added. At 150 days out, AUC 0.79.

While they were at it they found the other problem. One of the 3,400 automated features was mean sentiment across the account's support tickets—computed over *all* tickets, including ones filed after the renewal date. The tool had no concept of a prediction horizon, so it had cheerfully aggregated the future into the past. That single feature was worth about four points of the original 0.83.

The postmortem number was $6.2 million: ARR that walked out over two quarters in accounts which, scored by the two-feature model, had been flagged more than 140 days before anyone made a call.

Three thousand four hundred features, generated in an afternoon by a tool. Two features, generated in a hallway by a person who knew what the job actually was. The two won by a distance that isn't close.

---

Everything in Part IV was repair. Missing values, outliers, scales, categories—taking the data you were handed and making it safe to model, without lying to the model in the process.

This chapter is the other half of the work, and it is the half where models are actually won. A model can only combine the columns you give it. A gradient-boosted tree is very good at finding interactions, but only within the space your features span. If nobody ever computed "days since the champion logged in," there is no architecture, no hyperparameter, and no amount of data that recovers it. That column does not exist in the world until a human decides it should.

Which is why feature creation stays valuable in a period where everything else is getting commoditized. Anyone can download the same model your competitor is using. Nobody else has your operators' intuition about what makes a customer, a claim, or a shipment go bad.

---

## 14.1 Domain Knowledge: The Competitive Advantage

The best features in any production system almost always look obvious in retrospect and were invisible in advance. That asymmetry is the whole game, and it means your job is less about invention than about extraction—getting what an expert knows out of their head and into a column.

The people who have it are the ones doing the work: the CSM, the underwriter, the fraud analyst, the dispatcher, the floor supervisor, the radiologist. They are usually not in the room when features get designed, which is the actual root cause of most mediocre feature sets.

There is one question that works better than any other, and it is worth memorizing:

> *"When you look at one of these and you get a bad feeling before you can explain why—what did you just look at?"*

You are not asking for their model. You are asking what their eye went to. The answers come back as things like "I check whether the shipper changed their pickup window twice in a week," or "if the invoice address and the shipping city don't match I read it more carefully," or "when a claim gets reassigned to a third adjuster it's going to be a bad one." Each of those is a feature. Some of them are three lines of SQL.

Two other sources are already sitting in your organization and almost nobody mines them:

**The edge-case log from Chapter 7.** Every "wait, how do I handle *this* one?" that your labelers asked is a place where the phenomenon has structure your features don't capture. A labeling ambiguity and a missing feature are frequently the same fact viewed from two directions.

**Incident postmortems.** When a model was badly wrong and somebody wrote up why, the explanation is nearly always phrased in terms of a factor the model didn't have. That document is a feature request that nobody filed.

Where domain knowledge most often turns into math is **normalization against context**. A raw quantity usually means nothing on its own. A $50,000 transaction is unremarkable for one cardholder and an emergency for another. What carries the signal is the same number expressed relative to something the domain says is the right reference:

```python
df["amt_vs_own_median"] = df["amount"] / df["cardholder_median_90d"]
df["amt_vs_peer_median"] = df["amount"] / df["segment_median_90d"]
```

Almost every strong fraud, risk, and anomaly feature in production is some version of that ratio. The domain knowledge is not in the division. It's in knowing which denominator is the right one.

One caution, because domain knowledge is not automatically correct. Experts carry heuristics that were true five years ago, or that are true for the segment they personally handle and false elsewhere. Treat every expert-supplied feature as a hypothesis with a name attached, build it, measure it, and tell them what you found. The ones that don't hold up are worth reporting back—that conversation is usually more interesting than the ones that work.

---

## 14.2 Mathematical and Statistical Transformations

Chapter 12 reshaped existing features so algorithms could use them. This is a different activity that uses some of the same arithmetic: creating quantities that carry information no existing column carries.

**Ratios and rates** are the most productive and most neglected family. Any time you have two columns whose relationship means something the levels don't, the ratio is a candidate: revenue per employee, errors per thousand requests, claim amount over policy limit, storage cost over rows served. Models can theoretically learn a ratio from its components, and in practice they learn it badly—a tree approximates division with a staircase of splits, and a linear model can't do it at all.

Guard your denominators, because this is where new `inf` and `NaN` values enter a clean dataset:

```python
df["rate"] = df["numerator"] / df["denominator"].replace(0, np.nan)
```

Choosing `NaN` over `inf` is deliberate. A `NaN` flows into the missing-data machinery from Chapter 10, where you already have indicators and a policy. An `inf` propagates silently through means and standard deviations and poisons whatever it touches.

**Differences and deltas** encode change, which is frequently what actually matters. Current value versus 30 days ago. Days between events. Change since onboarding. A model handed only the current value has to infer motion from a snapshot.

**Log ratios** make growth symmetric. A change from 100 to 200 and from 200 to 100 are the same magnitude in opposite directions, but the raw ratios are 2.0 and 0.5, which no linear model reads as symmetric. `np.log(a / b)` gives you +0.69 and −0.69, and the transformations from 12.3 apply here for the same reason they applied there.

**Relative-to-own-history z-scores** turn a global measurement into a personal one:

```python
df["z_own"] = (df["value"] - df["entity_mean_90d"]) / df["entity_std_90d"].replace(0, np.nan)
```

That column answers "how unusual is this *for them*," which is almost always the question, and which Chapter 11 argued is the difference between an outlier and an error.

**Volatility features.** The rolling standard deviation of a value is a feature in its own right, and it is often more predictive than the value. Accounts whose usage bounces around are behaving differently than accounts with the same average and a flat line.

Polynomial and interaction terms belong to this family too, but they were covered in 12.5 and the combinatorial warning there still applies. Build the products the domain suggests; do not generate all of them and hope.

---

## 14.3 Aggregation and Window-Based Features

If you only take one section of this chapter into practice, take this one. Grouped time-window aggregations are the largest single source of predictive features in tabular machine learning, and they are also where the most expensive bug in this chapter lives.

The grammar is always the same: **an entity, a window, a statistic, a field.**

- entity: customer, merchant, device, account, sensor, driver
- window: 1 day, 7 days, 30 days, 90 days, since-signup, since-last-event
- statistic: count, sum, mean, max, std, nunique, slope
- field: amount, duration, error, login, ticket

Cross those and you have a large, principled candidate space. Name them so a stranger can read them—`merchant_txn_count_7d`, `account_distinct_admins_90d`—because in two years the naming convention is the only documentation that will still be accurate.

Multiple windows on the same quantity are how you encode acceleration. `count_7d` and `count_90d` individually say volume; their ratio says whether things are speeding up or falling off, and that ratio is usually the feature that ends up mattering.

Now the landmine. **A rolling aggregate that includes the current row has leaked.**

This is the most common leak in production feature engineering, it is one character wide, and it survives code review constantly:

```python
# Wrong. The window includes the current row, so every row
# has been told a little bit about itself.
df["mean_7d"] = df.groupby("account")["amount"].transform(
    lambda s: s.rolling("7D").mean()
)

# Right. Shift first: the window ends before this row exists.
df["mean_7d"] = df.groupby("account")["amount"].transform(
    lambda s: s.shift(1).rolling("7D").mean()
)
```

This is Chapter 9's point-in-time correctness in the place where you are most likely to violate it. The rule is the one that chapter gave you: a feature for a row timestamped *t* may only use data that existed strictly before *t*. Not before *t* plus a bit. Before *t*.

Two related traps in the same family:

**The label-window overlap.** If you're predicting a 30-day outcome, features must end at the prediction point, not at the label point. Aggregating over "the last 90 days" from *today* when the label was determined 30 days ago quietly folds the outcome period into the features.

**The retroactive-update column.** Some source fields get corrected after the fact—an order status that flips to `refunded` weeks later, a diagnosis code amended at discharge. Aggregating the current value of that field over a historical window uses information from the future even though every timestamp looks correct. Chapter 5's lineage work is what tells you which fields do this.

Last thing on aggregations, and it's an operational point rather than a statistical one: these features are expensive to compute at serving time. A 90-day aggregate over an entity's history is a query you cannot afford in a ten-millisecond budget, which is precisely why the feature stores in Chapter 9 exist. If you build a rich set of window features and no plan for serving them, you have built a very good offline model.

---

## 14.4 Feature Crosses and Combinations

A cross is the conjunction of two categoricals treated as one: not "country" and "device" as separate signals, but `country=BR AND device=android` as a distinct thing with its own behavior.

Crosses matter enormously for linear models, which cannot represent a conjunction at all—a linear model can learn that Brazil is riskier and that Android is riskier, but not that the combination is far riskier than either implies. Tree models get some of this for free by splitting on one feature and then the other, which is why crosses feel less essential in a gradient-boosting workflow. "Less essential" is not "unnecessary": an explicit cross saves the tree the depth it would spend rediscovering the conjunction, and depth is a budget.

The cost is cardinality, and it multiplies. Fifty countries crossed with 200 device models is 10,000 levels, most of which will appear a handful of times. You have just created exactly the problem Chapter 13 was about, on purpose, and you inherit the whole toolkit with it: frequency floors, rare bucketing, smoothed target encoding, hashing. Build a cross and you have signed up to encode it properly.

Which crosses should you build? The ones with a mechanism. If you can say a sentence explaining why the combination behaves differently than the parts—"weekend plus mobile plus a new payment method is how card testing looks"—build it. If the only justification is that both columns are in the table, you are generating candidates for a selection problem you haven't started yet.

For finding candidates rather than guessing, two practical approaches:

**Read a shallow tree.** Fit a depth-3 or depth-4 tree and look at which features co-occur along paths. The tree has already done interaction detection for you; the paths it found are crosses worth making explicit.

**SHAP interaction values.** More expensive, more rigorous, and it will rank pairs by how much their joint effect exceeds the sum of their individual effects. Use it on a sample; the full computation is quadratic in features.

The principled end of this spectrum is factorization machines and the wide-and-deep family, which learn low-rank representations of every pairwise interaction rather than requiring you to enumerate them. If you have very high cardinality and a lot of data, that machinery is doing what you would otherwise do by hand, better. Most teams don't need it, and the ones that do usually already know.

---

## 14.5 Automated Feature Engineering: Featuretools and Beyond

Automated feature engineering deserves a fair hearing, because the cold open above is not an argument against it. It's an argument against using it without knowing what it can and cannot do.

Deep feature synthesis, the idea behind Featuretools, works from a declared schema. You describe your tables and the relationships between them, and it applies aggregation primitives across relationships and transform primitives within them, stacking to a chosen depth. `MEAN(sessions.duration)` at depth one; `STD(users.MEAN(sessions.duration))` at depth two. It composes mechanically and it composes fast.

```python
import featuretools as ft

fm, defs = ft.dfs(
    entityset=es,
    target_dataframe_name="accounts",
    agg_primitives=["mean", "sum", "count", "std", "trend"],
    trans_primitives=["day", "month", "weekend"],
    cutoff_time=cutoffs,      # not optional
    max_depth=2,
)
```

That `cutoff_time` argument is the one that matters, and it is the one the team in the cold open didn't set. It tells the library, per row, the moment beyond which no data may be used. Without it, deep feature synthesis will happily aggregate across a row's entire history including everything after the label event, and it will produce a beautiful offline number while doing it. **Automated feature tools do not know what a prediction horizon is unless you tell them.**

Where automation genuinely earns its place:

- **Broad relational schemas.** Eight tables of nested relationships where hand-writing the aggregations would take a week and you'd miss half of them.
- **Early exploration.** Generating a wide candidate set to find out which *families* of features carry signal, before you invest in building any of them properly.
- **Combinations nobody would type.** Depth-two stacked aggregations are real features that human engineers rarely write out and sometimes should.

Where it structurally cannot help:

- **It can only recombine what is already logged.** This is the fundamental ceiling. Deep feature synthesis over the SaaS company's warehouse could generate ten thousand features and never produce "days since the highest-permission user logged in," because nothing in the schema marked that user as special. That fact lived in a human's head. The tool does the Cartesian product of the recorded world; it does not extend the recorded world.
- **Volume is a liability.** Three thousand four hundred features is a serving surface, a monitoring surface, and a maintenance surface. Every one of them is a column somebody has to compute in production, forever, at latency.
- **Interpretability drops off a cliff.** When a regulator or an executive asks why an account was flagged, `STD(users.MEAN(sessions.duration))` is not an answer anyone can act on.

The productive posture is to treat automation as a **candidate generator, not a feature set**. Run it wide, select hard—which is Chapter 15's entire subject—and then hand-build clean, well-named, cheaply-servable versions of the handful that survived. What you ship should be small enough that you can explain every column in it.

---

## 14.6 Feature Validation and Impact Assessment

A feature is not real because it was clever. It is real when it improves the metric you care about, at the decision horizon you actually have, without leaking.

**Ablation is the only real test.** Train with and without. For a small candidate set, leave-one-out tells you what each feature contributes on the margin. For a large one, add-one-in against a fixed baseline is cheaper and answers the question you're actually asking. Either way you get a number, and the number is frequently zero for features everyone was sure about.

**Test at the horizon you can act at.** This is the lesson of the cold open and it generalizes far past churn. A model evaluated at "time of renewal" and a model evaluated at "120 days before renewal" are different models with different useful features, and only one of them corresponds to a decision anybody can make. Ask what action the prediction triggers, ask how long that action needs, and evaluate there. A model that is accurate too late is a report, not a model.

**Be careful which importance you read.** Gain-based importance, the default in most gradient-boosting libraries, is biased toward high-cardinality features—a column with many distinct values gets more chances to produce a locally good split, so it accumulates gain whether or not it generalizes. Permutation importance computed on held-out data doesn't have that bias and answers a more useful question: how much worse does the model get when this column is scrambled. SHAP gives you direction and per-row attribution on top of that. None of the three tells you anything about causation, and all three get read as if they do.

**Run every new feature through the leakage checklist**, in this order, before you get excited:

1. Could this value have been known at prediction time? Not "does the timestamp look right"—could a system have computed it, then?
2. Is any input to it derived from the target, however indirectly?
3. Does its window extend past the label event?
4. Does the source field get retroactively updated?
5. Is it too good?

That last one is a heuristic and it is worth its weight. A single feature that produces an AUC above 0.9 on its own is leakage until proven otherwise. Chapter 6 put the same rule in the EDA smell test, and it has a better track record than most formal methods.

Two things nobody does that you should.

**Cost every feature at serving time.** A feature that adds half a point of AUC and requires a 90-day aggregation over an entity's full event history is not obviously worth shipping. Write the offline gain and the serving cost next to each other and let somebody make an actual decision.

**Retire features.** Feature sets accumulate and never shrink. Every column you keep is a permanent production dependency—a pipeline that can break, a schema that can drift, an unknown category that can arrive on a Tuesday. Once a quarter, ablate the bottom third of your feature set as a group. Most of the time the metric doesn't move, and you get to delete a third of your surface area. That is the highest-return hour of maintenance work available to a data team, and almost no one spends it.

---

## Quick Wins: Feature Ideas You Can Steal This Afternoon

**Ask one operator the bad-feeling question (30 min).** Find the person who works your model's subject matter by hand. Ask what they look at when something feels wrong before they can articulate why. Write down every answer verbatim. You will leave with three to five features, and at least one of them will not exist in any table you currently query.

**Add a ratio against the entity's own history (20 min).** Take your most important numeric column and divide it by that entity's 90-day rolling median, shifted by one. Add it, retrain, measure. This one feature earns its keep more often than any other single suggestion in this chapter.

**Add a second window to something you already aggregate (15 min).** If you have a 30-day count, add a 7-day count and their ratio. You have just given the model acceleration where it previously only had level.

**Grep your feature code for `.rolling(` without `.shift(` (10 min).** Every hit is a candidate point-in-time leak. Check each one against the row timestamp. This has a higher hit rate than anyone expects, including on code that has been in production for years.

---

## Your Homework

### Exercise 1: The Operator Interview (Time: ~1 hour, mostly listening)

Sit with someone who does your model's job manually and walk through ten real cases with them. Do not explain the model. Ask what they notice, in what order, and what would change their mind. Write down every quantity they mention, including the ones they mention dismissively. Then build the three cheapest ones and ablate them against your current model. Score this exercise on how many of the features you built were things nobody on the data team had considered—if the answer is zero, you interviewed the wrong person.

### Exercise 2: Find Your Own Leak (Time: ~45 minutes)

Pick your production model's five most important features by permutation importance. For each one, write a sentence describing exactly when its value becomes knowable, in wall-clock time relative to the prediction moment. Not the code path—the real-world moment the underlying fact becomes true. Any feature where you cannot write that sentence confidently is your next investigation, and any feature where the honest answer is "after" is a leak you are shipping today.

### Exercise 3: Move the Horizon (Time: ~1 hour)

Re-evaluate your model at twice the lead time it currently gets evaluated at. If it scores renewals at renewal, score them ninety days out. If it scores risk at transaction time, score it a week before. Compare the two feature-importance rankings. They will not be the same list, and the difference is the set of features you would need to build for the model to be useful early enough to act on—which is the version of the model your business actually wanted.

---

## Bridge to Chapter 15

You now have a machine for making features, and the immediate consequence is that you have too many of them. Every technique in this chapter multiplies: entities times windows times statistics times fields, crossed, ratioed, and stacked. A team that internalizes this chapter goes from forty columns to four hundred inside a month, and the model gets worse.

It gets worse for reasons that are worth naming precisely. Redundant features split importance between themselves and make everything look unimportant. Noisy features give a model additional opportunities to fit noise. High-dimensional spaces make distance meaningless in the way Chapter 11 described. And every column you keep is a permanent operational obligation.

Chapter 15 is the counterweight: filter, wrapper, and embedded selection methods; regularization paths that shrink features out of a model on their own; PCA and the modern nonlinear reductions and what they cost you in interpretability; and how to decide when a smaller model that a human can explain beats a larger one that scores half a point better.

---

*P.S. — The part of the churn story that stays with me is that nobody was wrong. The tool worked exactly as documented. The model was correctly trained and fairly evaluated. The 0.83 was real, on the data it was given, at the moment it was measured. Everything was fine except the one question nobody asked out loud, which is what a customer success manager is supposed to do with a prediction that arrives after the customer has already signed with someone else. Ask what the person on the other end is going to do with the number. Ask it first, before you build anything.*
