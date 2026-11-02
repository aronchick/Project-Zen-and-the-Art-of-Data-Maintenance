# Appendix D: Glossary and Terminology

Every concept this book introduces gets its full introduction once, in the chapter that is about it, and a short entry here. Use this appendix when you meet a word out of order. Each entry gives the plain meaning in one or two sentences and names the home chapter, where the concept arrives inside a story and gets the treatment it deserves. Entries are alphabetical; acronyms are expanded in D.2.

## D.1 Terms and Definitions

**Ablation.** Testing whether a feature is real by training the same model with it and without it and reading the difference in the metric you care about. The number is frequently zero for features everyone was sure about. Chapter 14.

**Active learning.** Training a quick model on a small labeled seed set and letting it tell you which unlabeled examples it is least sure about, labeling those, and repeating. You stop paying people to confirm what the model already knows. Chapter 7.

**Batch processing.** Working on a whole chunk of data at once, on a schedule, such as a nightly job that recomputes the day. The counterpart of streaming. Chapter 8.

**Cardinality.** How many distinct values a column can take. A device-type column with five values is low-cardinality; a placement column with 2.3 million is about as high as it gets. Chapter 13.

**Categorical variable.** A column whose values are labels drawn from a set (country, merchant, diagnosis code) rather than quantities you can add or average. There is no mean of a country. Chapter 13.

**Cohen's kappa.** Agreement between two labelers measured above what two people guessing would have produced, on a scale where 1.0 is perfect and 0 is chance. Raw agreement flatters; kappa does not. Chapter 7.

**Color space.** The definition of which physical color each channel value maps to. RGB(255, 0, 0) is a different red in sRGB and Adobe RGB, and a model can learn the wrong one. Chapter 16.

**Columnar storage.** Keeping every value of one column together on disk instead of every value of one row. Similar values compress together and a query reads only the columns it names. Chapter 3.

**Concept drift.** The inputs look the same but what they predict has changed, such as a fraud pattern the model learned that fraudsters have since abandoned. Compare data drift. Chapter 9.

**Correlation.** A score from minus one to one for how much two columns move together, and nothing more. It says nothing about which one causes the other or whether a third thing drives both. Chapter 6.

**Curse of dimensionality.** What happens when you have so many columns relative to rows that distances stop meaning anything, noise columns look like signal, and models overfit. Four problems wearing one name, with different fixes. Chapter 15.

**Data acquisition.** Accounting for who collected the data, how, under what agreement, and through which systems it traveled before it reached you. The trouble usually starts before anyone opens the file. Chapter 5.

**Data architecture.** The decisions about where data is stored, in what form, how it moves between systems, and which copy counts as the truth. Every pipe can be fine and the plumbing still be the bug. Chapter 8.

**Data contract.** A written, versioned agreement between a producing team and a consuming team about what will be delivered, in what shape, at what quality. A rename becomes a negotiation instead of an outage. Chapter 5.

**Data drift.** The distribution of the inputs moves while the feature is still computed correctly, such as average transaction size creeping up with inflation. Compare concept drift. Chapter 9.

**Data quality.** Whether the data is fit for the decision you are about to make with it. Not a property of the file; a property of the file and the decision together. Chapter 5.

**Data type.** The rule a system follows for what a value is and therefore what it may do with it: that a ZIP code is a label to match, not a number to add. Get it wrong and the system does arithmetic on a fact about the world. Chapter 2.

**Data warehouse.** A system built to load cleaned, typed tables and scan and summarize millions of rows fast with SQL. Governance and confidence come bundled, and so does a bill by the byte. Chapter 8.

**Deep feature synthesis.** Declaring your tables and the relationships between them and letting a tool (Featuretools) apply aggregation primitives across relationships and transform primitives within them, stacking to a chosen depth. It recombines what is logged; it cannot invent what is not. Chapter 14.

**Egress.** Paying a cloud provider to move data out of a region or off its network, having paid nothing to bring it in. The bill nobody budgets for. Chapter 4.

**Embeddings.** A model turning a text, an image, or a category into a long list of numbers arranged so that similar things land near each other. Entity embeddings do this for categorical values during training. Chapters 3 and 13.

**Encoding.** Turning each categorical label into one or more numbers a model can compute with. Every encoding scheme is a claim about the structure of the thing you are encoding. Chapter 13.

**Entity resolution.** Matching records that describe the same real-world thing across systems that never agreed on an identifier: John Smith, J. Smith Jr. and user_id_847291. Done badly, one customer becomes four. Chapter 5.

**Event time, ingestion time, processing time.** Three clocks: when the thing happened in the world, when your system first saw it, and when you got around to it. A model quietly using the wrong one predicts the past. Chapter 18.

**Exploratory data analysis.** Looking at a dataset's shapes, ranges, gaps and relationships before you build anything on it, to find out what is in there rather than what the documentation claims. A habit, not a kickoff task. Chapter 6.

**Feature.** One input column a model consumes, usually computed from raw data rather than read straight off it. Chapter 9.

**Feature cross.** Treating the combination of two categorical values as one new category with its own behavior, so a linear model can see a conjunction it cannot otherwise represent. The cost is cardinality, and it multiplies. Chapter 14.

**Feature engineering.** Reshaping raw columns into signals a model can learn from. The features that decide whether a model is useful usually do not exist in any source system until someone builds them. Chapters 1 and 14.

**Feature scaling.** Pushing every value in a column through the same shift and stretch so columns measured in different units end up on comparable rulers. The order and shape of each column survive; only the ruler changes. Chapter 12.

**Feature store.** Defining each feature once, computing it from that one definition, and handing the values to both training and serving. The two can no longer disagree because there are no longer two. Chapter 9.

**File format and serialization.** The agreement about how values are laid out as bytes so writer and reader see the same boundaries and types. Writing that agreement out is serialization; reading it back is deserialization. Chapter 3.

**Filter, wrapper, and embedded selection.** Three families of feature selection, sorted by how much they know about your model: filters score each column against the target on its own; wrappers run the model on candidate subsets; embedded methods let the model's own fitting drive features to nothing. Chapter 15.

**Gradient descent.** The fitting procedure behind neural networks and much of the field: start with guessed parameters, measure how wrong the model is, nudge every parameter a small step toward less wrong, and repeat. The size of the step is the learning rate. Chapter 12.

**Ground truth.** The set of labels you decide to treat as correct, the answer key the model is graded against. If the answer key is wrong, the scoreboard is wrong. Chapter 7.

**Hashing trick.** Mapping each category through a hash function into a fixed number of buckets and using the bucket number as the column. Fixed memory regardless of cardinality, at the cost of collisions and interpretability. Chapter 13.

**Imputation.** Replacing a missing value with an inferred one based on an assumption about why it is missing. It does not recover what was lost; it inserts a plausible substitute, and the plausibility rests entirely on the assumption. Chapter 10.

**Inter-annotator agreement.** Having several people label the same items and counting how often they concur. Chance agreement inflates the raw number; see Cohen's kappa. Chapter 7.

**Intrinsic dimension.** How many independent directions the data actually varies in, as opposed to how many columns it has. The gap between the two is every reduction technique's headroom. Chapter 15.

**Kappa architecture.** Deleting the batch layer and treating everything as a stream, replaying the same log through the same code when you need history. One implementation, one place to fix. Chapter 8.

**k-nearest-neighbors.** About the simplest model there is: to predict for a new row, find the k past rows whose feature values sit closest to it and go with what worked for them. "Closest" is a distance, which is why scaling decides what the model sees. Chapter 12.

**L1 regularization (lasso).** Penalizing the sum of absolute coefficient sizes so the weakest ones land on exactly zero and leave the model. Selection happens inside the fit. Chapter 15.

**Label and labeling.** The label is the answer attached to each training example; labeling is producing those answers, usually by humans, under a guideline. Chapter 7.

**Labeling guideline.** The written spec for your ground truth: what to do with sarcasm, mixed sentiment, and the case nobody anticipated, with worked examples. The gaps are where the bugs live. Chapter 7.

**Lambda architecture.** Running a slow, complete batch path and a fast, approximate streaming path side by side and merging them. It looks like the best of both; it is the business logic written twice. Chapter 8.

**Lineage.** The record of where a column came from and every transformation it passed through on the way, so that a bad number can be traced to its source. Chapter 1.

**Loss function.** The formula training minimizes, and therefore what "wrong" costs. Squared error lets one two-million-dollar claim outweigh a thousand fender-benders; change the formula, not the data. Chapter 11.

**Magic bytes.** The fixed signature at the start of a file that identifies its format, read instead of trusting the extension. `FF D8` is a real JPEG. Chapter 16.

**Mahalanobis distance.** A point's distance from the center measured in units of how the columns move together, so the metric knows height and weight travel as a pair. Chapter 11.

**Missing completely at random, missing at random, missing not at random.** Rubin's three answers to "why is this cell empty": no connection to anything; depends on columns you have; depends on the value you cannot see. The third is the trap. Chapter 10.

**Missingness.** The pattern of which cells are empty and why, read across the table rather than as a hole in it. Sometimes the shape is the answer. Chapter 10.

**Missingness indicator.** A yes/no column recording whether a value was present, kept next to whatever you do with the value itself. It turns a missing-not-at-random trap into an ordinary feature. Chapter 10.

**Nested cross-validation.** An inner loop picks a setting (how many features to keep, which hyperparameter) and an outer loop that never saw the choosing reports the score. Chapter 15.

**Normalization against context.** Expressing a raw number relative to the reference the domain says matters instead of reading it on its own: this transaction over this cardholder's own 90-day median. The knowledge is in the denominator. Chapter 14.

**Observability.** Instrumenting a pipeline so you can tell from the outside what it is doing inside: null rates and value ranges tracked next to the business metrics. You stop asking whether the dashboard is green and start asking whether to believe it. Chapter 1.

**One-hot encoding.** Giving every category its own yes/no column, exactly one of them hot per row. The correct default for most cases; its cost is width. Chapter 13.

**Outlier.** A value far from the rest of its column, or far from where the other columns say it should be. That is what every detector finds, and it is not a property of the number: whether it is an error or a fact is a claim about the process. Chapter 11.

**Out-of-fold computation.** Splitting the training data into folds and, for the rows in one fold, computing a target-derived feature using only the other folds, so a row's own label cannot reach its own feature. Chapter 13.

**Point-in-time correctness.** Computing each training row's features using only what existed at that row's moment. The rule behind feature stores, rolling windows, and most leakage you will meet. Chapter 9.

**Predicate pushdown.** Applying a filter before the data leaves disk rather than after it lands in memory, using the min and max stored with each chunk. Most of the file never gets read. Chapter 3.

**Provenance.** The actual, physical, organizational path the data took before it landed in front of you. Every step is a place it could have started lying. Chapter 5.

**Quality gate.** A check at the point of entry that blocks or flags rows failing the rules before they land in the warehouse. Prevention over detection over remediation. Chapter 5.

**Rare-category bucketing.** Taking every category below a frequency floor and calling it one explicit rare category. It bounds an unbounded vocabulary and gives unseen categories a home at serving time. Chapter 13.

**Regularization.** Adding a penalty to the training objective for coefficients that grow large, which is how you stop a linear model from fitting noise. The penalty is on magnitude, and magnitude depends on units. Chapter 12.

**Robust statistic.** A summary a few wild values cannot drag around: the median and the median absolute deviation instead of the mean and the standard deviation. Chapter 11.

**Schema.** The written-down list of every field a record carries and the type each one must be, with anything that does not match rejected. Chapter 1.

**Schema drift.** The upstream structure changing under you without anyone changing your code: a field renamed or retyped by a team that never heard of your model. Nothing errors; the numbers stop meaning what they meant. Chapter 2.

**Selection and reduction.** Two ways to have fewer columns: selection keeps a subset of the originals and discards the rest; reduction replaces them with a smaller number of constructed mixtures nobody can walk to. Chapter 15.

**Sentinel value.** A placeholder parked in a column to mean "unknown" that looks exactly like a measurement, such as an age of 999. One of these in a mean shifts every customer's age. Chapter 2.

**SHAP values.** Splitting each individual prediction into one contribution per feature so the pieces add up to the prediction. Interaction values do the same for pairs. Chapter 14.

**Smoothing toward the prior.** Pulling every category's own average toward what you would believe before seeing a single row of it (the global average), weighted by how much evidence backs it. Chapter 13.

**Stability selection.** Rerunning your feature selection on many subsamples and reporting how often each feature survives, not just whether it did. Chapter 15.

**Streaming.** Processing each event as it arrives so the answer is seconds old. The counterpart of batch. Chapter 8.

**Structured and unstructured data.** Structured data sits in named, typed fields a machine can read directly; unstructured data needs something to extract the meaning first. Most real data is in between, which is where the trouble is. Chapter 2.

**Target encoding.** Replacing each category with a number summarizing how that category has behaved, such as its average click-through rate. Powerful, and the most reliable way there is to leak labels into features. Chapter 13.

**Target leakage.** A feature carrying information you will not actually have at prediction time, because it came, directly or by a side door, from the thing you are predicting. It does not look like a bug; it looks like success. Chapter 6.

**Technical debt.** Taking a shortcut today and paying interest on it in every bug, workaround and rewrite it causes. Bad data decisions compound the same way. Chapter 4.

**Three-sigma rule and z-score.** Flagging any value more than three standard deviations from the mean, because under a bell curve that happens one time in 370. The number of standard deviations a value sits from the mean is its z-score; the rule rests entirely on "under a bell curve". Chapter 11.

**Train/serve skew.** The feature values a model learned from and the ones it is fed in production quietly disagreeing: same name, different number. Chapter 9.

**Transformation.** Pushing a column through a function that changes its distribution, not just its range, such as a logarithm on claim severity. Scaling moves and stretches a distribution; transformation changes its shape. Chapter 12.

**Type inference.** Letting a library guess each column's type from what the values look like: five digits, so "integer", and a ZIP code becomes a number. Chapter 2.

**Univariate F-score.** Scoring each column on its own by how far one group's mean sits from the others', ignoring every other column. The classic filter, and the classic way to find noise that looks like signal. Chapter 15.

**Unsupervised.** Learning the shape of normal from the inputs alone, with no labels saying which rows were actually bad. It answers "is this unusual", never "is this wrong". Chapter 11.

**Validation set.** A slice of labeled examples the model never trains on, held back so you can grade it. If the answer key is wrong, the scoreboard is wrong. Chapter 1.

**Weak supervision.** Training on labels that are cheap and imperfect, produced by rules, heuristics or other models, while modeling how unreliable each source is. Little ground truth goes a long way. Chapter 7.

**Windowing.** Grouping a stream of events into time boxes and computing over the box instead of the raw events: orders per five minutes instead of every click. Chapter 18.

**Winsorizing.** Pulling every value above a chosen percentile down to that percentile's value (and likewise below). A cap that keeps the row and discards the extreme. Chapter 11.

## D.2 Acronyms and Abbreviations

**AUC.** Area under the ROC curve: the probability that a yes/no predictor ranks a random positive above a random negative. 0.5 is a coin flip, 1.0 is perfect. Chapter 6.

**CTR.** Click-through rate. Chapter 13.

**CV.** Cross-validation: scoring a model on slices of the training data it was not fit on. Chapter 12.

**EDA.** Exploratory data analysis. Chapter 6.

**ETL.** Extract, transform, load: the pipeline shape that moves data from source systems into a warehouse. Chapter 4.

**KS test.** Kolmogorov-Smirnov test: a statistical test for whether two samples came from the same distribution, used for drift detection. Chapter 5.

**MAD.** Median absolute deviation, the robust cousin of the standard deviation. Chapter 11.

**MCAR, MAR, MNAR.** Missing completely at random, missing at random, missing not at random. Chapter 10.

**MSE.** Mean squared error, the usual training objective for a regression model. Chapter 11.

**OLAP.** Online analytical processing: the scan-and-summarize workload a data warehouse is built for. Chapter 8.

**PCA.** Principal component analysis: replacing columns with a smaller set of constructed directions of maximum variance. Chapter 15.

**SHAP.** SHapley Additive exPlanations; see SHAP values. Chapter 14.

**SLA.** Service-level agreement: a promise about freshness, availability, or quality that a data producer makes to a consumer. Chapter 5.

**SMOTE.** Synthetic minority oversampling technique, a way to manufacture examples of a rare class. Chapter 21.

**TF-IDF.** Term frequency times inverse document frequency: a weighting that scores a word by how often it appears in a document and how rare it is across documents. Chapter 15.
