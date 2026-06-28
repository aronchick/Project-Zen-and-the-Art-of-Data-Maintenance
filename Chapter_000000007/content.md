# Chapter 7: Data Labeling and Annotation

## Or: Your Ground Truth Is a Rumor

A medical-imaging startup I consulted for spent fourteen months and a little over $2 million building a model to flag a specific abnormality on chest X-rays. Their held-out test set hit 94% accuracy, beat the published benchmarks, and sailed through the demo that closed their Series A. Then they ran it against a fresh batch of images read by a panel of three radiologists, and accuracy fell off a cliff—down into the seventies, with a false-negative rate that would get someone killed.

Nothing had changed; what happened was simpler and much more embarrassing: somebody finally checked the labels.

The original training set had been labeled by contractors—medical, credentialed, not cheap—working through a backlog of 180,000 images on a per-image rate. But two things had quietly corroded the labels. First, the annotation guideline for the abnormality had been revised once, about seven months in, and nobody re-labeled the images annotated under the old definition. So roughly a third of the training set was graded by one rubric and the rest by another, with the same column name and the same `0`/`1` values. Second, and worse, when the team pulled a sample of 500 images and had all three radiologists re-read them blind, the radiologists agreed with each other only about 70% of the time on the hard cases. The "ground truth" the model had been so carefully trained to reproduce wasn't truth. It was one tired contractor's opinion, captured at a per-image piece rate, under a definition that had changed midstream.

The model had learned the labels perfectly; the labels were the problem.

We spend Chapters 1 through 6 worrying about where data comes from, what type it is, how it's stored, what it costs, and how to investigate it. All of that assumes the *target*—the thing you're teaching the model to predict—is solid. In so MANY cases, it isn't. Your labels are data too, with all the same diseases: they're inconsistent, they drift, they're produced by upstream humans with their own bugs and "temporary" definitions that became permanent. And worse, many of the techniques you're learning in this book can overlook label issues, because maintaining them require a different pipeline. And not paying attention will have significant downstream impacts; bad features make a model a little worse, and bad labels put a ceiling on how good it can *ever* be. You cannot train your way past a label that's wrong; the model will learn the error faithfully and report it back to you as accuracy.

---

## 7.1 Label Consistency: The Foundation of Model Performance

The base case for bad models is bad. REALLY bad. If 10% of your labels are wrong, the best possible model—a hypothetical oracle that perfectly captures every real pattern in the features—will still appear to be 10% wrong, because it'll disagree with the bad labels exactly where the bad labels are bad. Your measured accuracy is capped by your label accuracy. Worse, the model doesn't just eat the error; it learns from it, fitting itself to noise and getting confidently stupid in the exact regions where your labelers were confused.

Consistency matters more than almost anyone budgets for, and there are two kinds, both of which will hurt you.

**Inter-annotator inconsistency** is when two people label the same example differently. Show ten people a borderline product review and ask "positive or negative?" and you will get a split, every time, because "this product is fine, I guess" is genuinely ambiguous and people resolve ambiguity differently. The 70% radiologist agreement from the opening wasn't incompetence; it was the irreducible disagreement of experts on hard cases, and if your guidelines don't tell people how to resolve it, every labeler resolves it their own way and your target column becomes a blend of a dozen private rubrics.

**Intra-annotator inconsistency** is sneakier: the *same* person labels the same example differently on Tuesday than on Friday. Fatigue, drift, a guideline they half-remember, the slow recalibration that happens when you've stared at 400 examples and your sense of "normal" has wandered. Slip the same image into a labeler's queue twice, a week apart, and measure how often they agree with *themselves*. People are routinely shocked. If a labeler can't reproduce their own judgment, the model certainly can't.

The root cause of most inconsistency isn't bad labelers. It's a bad **labeling guideline**—or none at all. The guideline is the spec for your ground truth, and like any spec, the gaps are where the bugs live. "Label the sentiment of this tweet" is not a spec; it's a wish. A real guideline says what to do with sarcasm, what to do with mixed sentiment, what to do with a tweet that's negative about a competitor, what counts as the "subject," and it does it with worked examples and explicit edge cases. Writing that document is the single highest-leverage hour in the entire labeling effort, and it's the one everybody skips because it feels like overhead next to the "real" work of cranking through images.

It is not overhead. It is the work.

Put a number on it, because numbers are what move people. The medical startup re-labeled after the disaster. Cleaning the training set—reconciling the two guideline eras, re-reading the ambiguous third with a three-radiologist panel and adjudication—ran about four months and a few hundred thousand dollars on top of the original spend. A two-page guideline and a kappa check at month one would have cost a single senior radiologist maybe a week. That's the trade the whole chapter is about: a week of defining the target up front, against four months and a redo at the end, against the version where nobody catches it and the false negatives ship to a hospital. The cost of label inconsistency isn't linear and it isn't optional. It's deferred, it compounds, and it comes due at the worst possible time—usually in front of a customer or a regulator.

---

## 7.2 Annotation Strategies: In-House, Crowdsourcing, and Programmatic

Once you accept that labels are a product you're manufacturing, the question becomes how to manufacture them. There are three broad approaches, and the right answer is usually a combination staged over the life of the project.

**In-house expert labeling** is what the medical startup did—domain specialists, on payroll or contract, doing the work directly. You get the highest ceiling on quality because the people understand the domain, and you get a tight feedback loop for refining the guideline. You also get the highest cost and the lowest throughput. Credentialed radiologists do not label 180,000 images quickly or cheaply, and the more expert your labelers, the more expensive every hour of inconsistency becomes. In-house is right when the task genuinely requires expertise and when getting it wrong is expensive—medical, legal, financial, safety. It's the wrong tool for "is there a cat in this picture," where you're paying specialist rates for a kindergarten task.

**Crowdsourcing**—Mechanical Turk, Scale, Labelbox's crowd, and the rest—flips the trade-off. Massive throughput, low per-label cost, and a quality floor that depends entirely on how well you've designed the task and the consensus mechanism. The naive version, one anonymous worker per item at a few cents each, produces garbage, because you've combined the world's least-contextualized labelers with the world's vaguest instructions. The version that works leans on **redundancy**: send each item to three or five workers and take the majority vote, which turns individually-noisy labelers into a collectively-decent signal—as long as the errors are random rather than systematic. (If the task is *consistently* confusing in the same direction, all five workers make the same mistake and consensus launders the error into false confidence. Consensus fixes noise, not bias.) Crowdsourcing is right for high-volume tasks where a clear guideline can make a non-expert as good as an expert, which is a larger set of tasks than snobs admit and a smaller set than vendors promise.

**Programmatic labeling** is the most leveraged and the most dangerous: you write code—rules, heuristics, pattern matches, or an existing model—to generate labels at scale. "Any transaction over $10,000 to a new payee in a different country gets flagged." "Any review containing 'broke after one use' is negative." You can label ten million examples in the time it takes a human to do ten. The catch is that you've now encoded your *assumptions* as ground truth, and every blind spot in your heuristic becomes a blind spot in your model, multiplied by ten million. Programmatic labeling shines as a *first pass*—get rough labels cheaply, then spend your expensive human hours auditing and correcting rather than starting from a blank slate. Section 7.5 is where this idea grows up into weak supervision.

Run the economics and the staging strategy writes itself. Say you have a million items to label. Pure in-house experts at, conservatively, $1 to $4 per careful label (a credentialed specialist reading at a few hundred items a day adds up fast) puts you somewhere between $1 million and $4 million and a timeline measured in quarters. Pure crowdsourcing at three redundant workers per item and a few cents a worker lands closer to $30,000-$150,000 and a timeline in weeks—but only on tasks where a non-expert with a good guideline is actually competent, which excludes the medical case entirely. Programmatic is nearly free per label and nearly instant, and worth precisely what your heuristics are worth.

The mature pattern stages all three to arbitrage that 100x cost spread. Programmatic for a cheap first pass that catches the easy 80% and shrinks the human queue by five-fold. Crowdsourcing with consensus for volume on the medium cases. In-house experts reserved for the hard cases, the edge cases, and—critically—for *auditing the other two*: a few thousand expert-labeled items, salted into the cheaper pipelines, that tell you whether the cheap labels are any good. Spend your most expensive people where their judgment actually changes the outcome, not where a rule or a crowd would have gotten it right anyway. The startup from the opening inverted this exactly—it paid specialist rates to grind the *whole* backlog, including the trivially-easy negatives, and still skipped the audit step that would have caught the guideline change. Most expensive labelers, least leveraged deployment.

---

## 7.3 Quality Control: Inter-Annotator Agreement and Validation

You measure what you manage, and "the labels seem fine" is not a measurement. The standard instrument for label quality is **inter-annotator agreement**: have multiple people label the same items and quantify how often they concur. The naive version is raw percent agreement—"they agreed on 85% of items"—and it's misleading, because some agreement happens by pure chance. If 95% of your examples are the negative class, two labelers who both just guess "negative" every time will agree 90% of the time while contributing exactly nothing.

**Cohen's kappa** corrects for that. It measures agreement *above what chance would produce*, on a scale where 1.0 is perfect, 0 is no better than random, and—yes—negative values are possible when annotators are below random, which happens and is as bad as it sounds. **Fleiss' kappa** extends the same idea to more than two annotators. You don't need to memorize the formula; you need to internalize the interpretation. The rough field convention: above 0.8 is strong, 0.6 to 0.8 is workable, 0.4 to 0.6 should worry you, and below 0.4 means your labelers are essentially disagreeing and your "ground truth" is a coin flip wearing a lab coat. The medical startup's 70% raw agreement on hard cases almost certainly translated to a mediocre kappa once you backed out the easy negatives—a number that, had anyone computed it before training, would have screamed "fix the guideline before you spend two million dollars."

Two practices turn this from a one-time audit into an actual control system. First, **gold standard items**: a set of examples labeled carefully by your best experts, with an agreed answer, salted invisibly into every labeler's queue. A common ratio is one gold item in every fifteen or twenty—enough to track each labeler's accuracy in near-real-time without burning much of the budget on already-answered questions. Now you can measure each labeler against truth continuously, catch the one whose agreement quietly slid from 0.9 to 0.7 over a long afternoon, pause them before they corrupt another thousand items, and retire the gold questions that turn out to be ambiguous (if your experts can't agree on a gold item, it was never gold—it's an edge case in disguise, and it belongs in the log from 7.6). Second, **adjudication**: a defined process for what happens when labelers disagree, rather than silently averaging the conflict away. Route disagreements to a senior labeler, and treat every adjudicated case as a bug report against the guideline—because a disagreement is usually not two people being dumb; it's two people hitting a case the spec didn't cover. The disagreements are a gift. They're showing you, for free, exactly where your definition of the target is underspecified. This is the same instinct as the EDA from Chapter 6: the anomalies are where the information is.

---

## 7.4 Active Learning and Smart Labeling Strategies

Labeling is expensive, so the obvious question is: do you need to label everything? Almost never. Most datasets are wildly redundant—the ten-thousandth clear-cut example teaches the model nothing it didn't learn from the first hundred. The expensive, valuable examples are the hard, ambiguous, near-the-boundary ones, and they're a small fraction of the whole.

**Active learning** is the discipline of spending your labeling budget where it matters. The loop is simple: label a small seed set, train a quick model, then let that model tell you which *unlabeled* examples it's most confused about—the ones where it's predicting 51/49, sitting right on the decision boundary. You label those, retrain, and repeat. Instead of labeling 100,000 random examples, you might get the same model quality from 15,000 well-chosen ones, because you stopped paying people to confirm what the model already knew and started paying them only for the cases that actually move the boundary.

The dollars follow directly. At $2 a label, 100,000 random labels is $200,000; the 15,000 the model actually needed is $30,000. Same accuracy, $170,000 saved—and faster, because you labeled a sixth of the volume. That's not a rounding-error optimization; that's the difference between a project that gets funded and one that doesn't. The selection criterion can get more sophisticated than raw uncertainty (you can also chase examples the model finds *surprising*, or ones representative of big unlabeled clusters), but plain "label what the model is least sure about" captures most of the gain and is a half-day to implement.

The savings are real—often a 3-5x reduction in labels for equivalent performance—but there are two traps worth naming. The first is the **cold start**: the model's notion of "confusing" is only as good as the model, and early on the model is bad, so its first few rounds of "I'm confused about these" can be noisy. You ride through it with a decent random seed set before you start trusting the model's requests. The second is **sampling bias**: by definition, active learning builds a training set skewed toward the boundary, which is great for the model and terrible for your held-out metrics if your *test* set is drawn the same way. Keep a separate, randomly-sampled, honestly-labeled test set that active learning never touches, or you'll measure yourself against the same boundary-heavy distribution you trained on and get a number that flatters you. (This is the Chapter 6 sampling lesson in a new costume: a convenient sample is a lying sample.)

---

## 7.5 Weak Supervision and the Snorkel Framework

Active learning makes human labeling efficient. Weak supervision tries to get most of the way there with far less human labeling at all, and it's the most important idea in this chapter for anyone facing a million unlabeled examples and a budget for ten thousand.

The premise: instead of one perfect, expensive label per example, write a bunch of cheap, noisy, individually-mediocre **labeling functions** and combine them statistically. A labeling function is just a heuristic that votes—a keyword match, a regex, a lookup against an external list, the output of an off-the-shelf model, a rule a domain expert dictated in a meeting. Each one is wrong a meaningful fraction of the time. Each one covers only part of the data and abstains on the rest. Individually, none is good enough to ship. The bet is that *collectively*, if you model how much they agree and disagree, you can estimate a probabilistic label far better than any single function—and do it across your entire unlabeled dataset for the cost of writing a few dozen rules.

Make it concrete. Say you're flagging customer-support tickets that are *urgent*, and you have 800,000 unlabeled tickets and budget to hand-label maybe 2,000. You write labeling functions: one votes "urgent" if the text contains "down," "outage," or "can't log in"; one votes "urgent" if the customer's account tier is enterprise; one votes "not urgent" if the ticket is tagged as a feature request; one votes "urgent" if there are three or more messages in the last hour; one defers to an off-the-shelf sentiment model and votes "urgent" on strongly negative tickets. Each of these is wrong constantly—"can't log in to tell you how much I love this" is not urgent, plenty of enterprise tickets are idle questions—and each one abstains on most tickets. But where several fire together, confidence climbs, and the disagreements are themselves informative. That's the raw material the next step refines.

**Snorkel**, which came out of the Stanford AI Lab, is the framework that made this rigorous. Its key trick is that it learns the *accuracies* of your labeling functions without ground truth, by looking at their agreement structure: a function that constantly disagrees with the consensus is probably unreliable and gets down-weighted; functions that correlate get their shared error discounted instead of double-counted. The output isn't hard labels but probabilistic ones, which you feed to a model that can learn from soft targets. A team can stand up a labeled training set over a domain in days instead of the months hand-labeling would take—and, more importantly, when the problem definition changes, you *edit the rules and regenerate*, instead of re-hiring an army to re-label from scratch. That maintainability is the real prize. Your labels become code: versioned, diffable, reviewable, and cheap to rerun.

It is not magic. If all your labeling functions share a blind spot—if they're all fooled by the same kind of example—weak supervision will confidently propagate that blind spot, exactly like the programmatic-labeling failure from 7.2, because it *is* that failure with better statistics. Weak supervision multiplies the leverage of your domain knowledge, including the leverage of your domain knowledge being wrong. You still need a small, clean, human-labeled validation set to tell you whether the whole apparatus is producing anything trustworthy. There's no escaping the need for *some* real ground truth; weak supervision just lets a little of it go a very long way.

---

## 7.6 Edge Cases Documentation and Management

Every labeling effort generates a stream of "wait, how do I handle *this* one?" Those questions are the most valuable output of the entire process, and most teams throw them away—answered in a Slack thread, in a DM, in a hallway, and never written down. Then the next labeler hits the same case, makes a different call, and your consistency quietly erodes one undocumented edge case at a time.

The fix is mundane and it works: a living **edge-case log**, attached to the guideline, that records the weird case, the decision, and the *reasoning*. "Product review that's positive about shipping but negative about the product → label by the product, not the experience, because the model predicts product quality. (2024-03-12)" Every entry is a patch to your ground-truth spec. Over a few weeks the log becomes the most valuable document in the project—the accumulated case law of what your labels actually mean—and onboarding a new labeler becomes "read the guideline and the edge-case log" instead of "absorb six months of tribal Slack history you'll never see."

There's a second reason the log matters: edge cases are where definitions silently fork. A content-moderation team I know labeled "violent content" for two years with no written rule for cartoons. Half the labelers called animated violence a violation; half didn't; nobody noticed because no two of them ever reviewed the same item. The split only surfaced when the model started behaving erratically on animation and someone finally pulled the labels apart by labeler. Two years of training data carried two incompatible definitions of the target fused into one column—and the only fix was to define the rule that should have existed on day one and re-label everything animated. An edge-case log would have surfaced that fork in week one, as a single logged question with a single ruling, for the price of someone bothering to write it down.

This connects directly to problems you'll meet later. The edge cases are very often your rare classes—the fraud, the defect, the disease, the 0.1% that's the entire point of the model—which means they're also where class imbalance bites (Chapter 21) and where a single mislabel costs you disproportionately because you have so few examples to begin with. Mislabeling one of fifty fraud cases is a 2% corruption of the only signal that matters. The discipline of documenting edge cases isn't bureaucratic tidiness; it's how you protect the scarce, expensive, high-stakes labels that determine whether the model works at all.

---

## Quick Wins: What You Can Do Monday Morning

**Run a relabeling spot check (1 hour).** Pull 50 random examples from your training set and re-label them yourself, blind to the existing labels. Then compare. If you disagree with your own production labels on more than a handful, you've just found the ceiling on your model's accuracy—and an excellent reason to look harder before you tune another hyperparameter.

**Compute one real agreement number (1 hour).** Take any 100 examples, have a second person label them independently, and calculate Cohen's kappa (it's three lines with `sklearn.metrics.cohen_kappa_score`). A kappa under 0.6 means your guideline, not your model, is the thing to fix this quarter.

**Write the guideline you assumed existed (2 hours).** For your most important label, write down the definition with three worked positive examples, three negative, and three genuine edge cases with rulings. Hand it to someone unfamiliar and watch where they hesitate. Their hesitations are your spec's bugs.

**Start the edge-case log (15 minutes).** Make the document. Put one entry in it. The hardest part of a living document is creating the file; do that today.

---

## Your Homework

### Exercise 1: Measure Your Own Consistency (Time: ~45 minutes)

Take 30 examples you labeled at the start of some past project. Re-label them now, cold, without looking at your originals. Compute how often you agree with your past self. Write down the cases where you disagreed and *why*—usually it's a definition you've since refined in your head but never wrote down. That gap, between the rule in your head and the rule on paper, is the gap your whole team is falling into.

### Exercise 2: Audit a Labeled Dataset You Trust (Time: ~1 hour)

Pick a labeled dataset you consider "clean"—bonus points for a public benchmark. Pull 50 examples and scrutinize the labels against the dataset's own documentation. Known benchmarks have been shown to carry meaningful label-error rates in their test sets; your in-house data is not magically cleaner. Document what you find, and notice how it reframes every leaderboard number you've ever trusted.

### Exercise 3: Build a Weak-Supervision Prototype (Time: ~2 hours, optional)

Take an unlabeled (or label-stripped) text dataset and write five labeling functions—keyword rules, regexes, length heuristics, whatever. Combine them with majority vote first, then with Snorkel's label model, and compare both against a small hand-labeled validation set. Feel where the labeling functions' shared blind spots show up. You'll never again confuse "I wrote rules" with "I have ground truth."

---

## Bridge to Chapter 8

You now have data you've sourced (Chapter 5), investigated (Chapter 6), and labeled with something resembling discipline (this chapter). It's sitting in files, in notebooks, in a database somebody set up two years ago. The next problem is architectural: where does all of this actually *live*, how does it flow from raw ingestion to model-ready, and what happens to your carefully-labeled data when "the dataset" outgrows a laptop and becomes a system? Chapter 8 is about the architecture patterns—warehouses, lakes, lakehouses, and the real trade-offs nobody tells you—that determine whether your data stays trustworthy at scale or quietly rots in a swamp.

---

*P.S. — The dirty secret of every leaderboard is that the test set has wrong answers in it too, which means somewhere out there a model is being penalized for being right, and a worse model is winning because it learned to reproduce the same mistakes the labelers made. Somewhere, a state-of-the-art result is just two annotators sharing a hangover. Sleep well.*
