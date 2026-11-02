# Style guide

Standing editorial rules for every chapter. Apply them to new writing and to any chapter you touch.

## 1. Show the code next to the technique

Whenever the text names a technique a reader could actually run (a transform, an estimator, a statistic, an encoder, a test, a storage or file-format operation), the Python that executes it sits right there: in the same paragraph or immediately after it. One line is enough.

- Use the most widely used Python library for that technique: scikit-learn for preprocessing, models and metrics; pandas for tabular work; numpy for math; scipy.stats for statistical tests; PyArrow for Arrow and Parquet. Name a less common library (category_encoders, Snorkel, Feast, Great Expectations, ydata-profiling) the first time it appears in a chapter.
- Form: inline backticks for a single call, e.g. `np.log1p(x)` or `PowerTransformer(method="yeo-johnson")`. A fenced Python code block of at most about eight lines when imports or several steps are needed.
- The code is illustrative, not a tutorial. It should be the shortest honest way to do the thing, and it must be correct for the current library API. Verify any call you are not certain of by running it: `uv run --with scikit-learn --with pandas --with scipy python -c "..."`. Never invent an API.
- Show a technique's code once per chapter, at its main mention. Later mentions in the same chapter do not repeat it unless they need a different variant.
- This applies to techniques, not to concepts. "Data contracts" or "the lakehouse pattern" get no snippet; "Cohen's kappa" or "KNN imputation" do. Do not force code into philosophy or architecture prose where nothing is runnable.

Worked example from Chapter 12.3. Before the rule, Box-Cox and Yeo-Johnson were described in prose and the reader was told "in scikit-learn both live in PowerTransformer". After the rule, each named technique carries its call at the point it is named:

> **The log transform** ... Use `np.log1p(x)` (log of 1+x) so that zeros survive.
> **Box-Cox** generalizes this into a family parameterized by λ; `PowerTransformer(method="box-cox")` estimates λ for you. It requires strictly positive input ...
> **Yeo-Johnson** is Box-Cox extended to handle zeros and negatives: `PowerTransformer(method="yeo-johnson")`, which is also scikit-learn's default ...

## 2. Define the term before you lean on it

Write for a reader who is smart but new to the field. The goal of the book is to educate people who may not have the context yet.

- When a chapter is about a concept (scaling, encoding, imputation, a feature store, exploratory data analysis), the concept gets a plain-English definition in the first few paragraphs of the body, before any numbered section uses it as a given. The cold-open disaster story may run first; the definition lands before section X.1 depends on it.
- Inside sections, the first time the argument leans on a term the reader may not have (k-nearest-neighbors, regularization, AUC, gradient descent, one-hot, cardinality, train/serve skew, MCAR), give it one or two sentences of definition at that point, in the voice of the book, not as a glossary aside or a bulleted definitions block.
- Do not define words any technical reader already has (mean, column, API, database, CSV).
- Test for every paragraph: could someone who has never trained a model follow this sentence? If not, define or cut the term.

**Introducing a concept for the first time.** When a concept enters the book for the first time, the reader is meeting a thing, not recalling a word. Introduce it inside the narrative, in this order:

1. Arrive from the problem. The story or argument has just posed a need, and the concept appears as a response to it: "A common solution for handling this is something called ..."
2. Name it in quotes, as a thing: something called "one-hot encoding".
3. Say what you do with it in a plain "where you ..." clause: an action from the practitioner's seat, in everyday words, not a textbook definition.
4. Give the story's own instance in parentheses: (in this case, the average click-through rate observed for that placement).
5. Close with the payoff in one short sentence.

Only after that does the term get used as a given. The compressed em-dash appositive ("one-hot encoding—one yes/no column per category—") stays available for minor terms later in a chapter; a chapter's central concepts get the full introduction at their first appearance.

Worked example from the Chapter 13 cold open. Before:

> One-hot encoding 2.3 million categories—one yes/no column per category—was obviously off the table, so an engineer reached for target encoding. Replace each placement with the average click-through rate (CTR) observed for that placement. One column instead of 2.3 million. Directly, obviously informative.

After:

> A common solution for handling this is something called "one-hot encoding", where you force a code path or machine learning model to make a constrained categorization within a small set of choices. However, one-hot encoding 2.3 million categories was obviously off the table, so an engineer reached for target encoding, where you replace each placement with information about the category to aid placement (in this case, the average click-through rate observed for that placement). One column instead of 2.3 million, obviously informative.

## 3. No version pins; uv for all Python

The code targets the current release of every library, and the reader runs it without setting up an environment first.

- Never name a specific library or Python version anywhere in the book: no "added in 1.3", no "deprecated in 1.8, gone in 1.10", no "pandas 3.1", no pinned requirements. If a behavior is version-dependent, describe the current behavior and say nothing about older releases.
- Every Python invocation the reader sees runs through uv: `uv run --with <libs> python script.py`, `uv run --with <libs> python -c "..."`, or `uvx <tool>` for command-line tools such as dvc. Never pip install, bare python, conda, poetry or venv. A script the reader is meant to save carries PEP 723 inline metadata (`# /// script` with a `dependencies = [...]` list, no version specifiers) so plain `uv run script.py` resolves everything.
- Appendix B, section B.0, is the one place that explains uv; chapters only point there.

## 4. Existing conventions

- No first-person experience claims ("I once saw", "in my experience"). Earlier passes removed these book-wide; do not reintroduce them.
- Preserve the conversational cold-open voice: every chapter opens with a disaster story, the tone is dry and direct, em-dashes are house style, and the two rules above add to the prose rather than restyle it.
