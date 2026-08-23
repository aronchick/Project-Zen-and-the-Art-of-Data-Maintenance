# Chapter 8: Data Architecture Patterns

## Or: How Two Correct Numbers Started a War

An e-commerce company had one number that ran the place: live revenue. Gross merchandise value, updating in near-real-time on a wall of dashboards that the growth team, the ops team, and the CEO all watched like a heartbeat—the number that decided whether a flash sale was working, whether to pour another hundred grand into an ad campaign, whether today was a good day. It was the most-watched number in the building, and they computed it twice.

They didn't mean to compute it twice. It happened the way these things always happen. The real-time path was a Flink job that summed orders as they streamed in, because the growth team cannot wait until tomorrow morning to find out whether the sale they launched at noon is working. The batch path was a nightly Spark job that recomputed the day's revenue from scratch because finance needed clean, complete, reconciled numbers for the books, investor updates, and the monthly deck for the board. Same logic, two codebases, two languages, two teams. For a while, the two numbers agreed closely enough that nobody looked.

Then somebody found a bug in how orders were bucketed at the day boundary—an order placed at 11:59:59 landed in the wrong day, an off-by-one at the window edge. They fixed it in the streaming job, shipped it, closed the ticket, and felt good. But nobody fixed it in the batch job, because nobody remembered the batch job *also* bucketed orders by time, in its own slightly different way, in a file owned by a different team, three Slack channels over.

So now the two numbers diverged, JUST enough. The live dashboard said one thing during the day; the nightly recompute said something a little different the next morning. The growth team worked off the real-time number. Finance and the board deck worked off the batch number. For six weeks, two groups of smart people made confident decisions based on two different sources of truth, and every meeting where they compared notes turned into a low-grade argument over whose dashboard was broken. Both numbers were *correct*, given the code that produced them. That was the problem.

The reconciliation project—the forensic accounting to figure out which days' revenue had been computed under which version, and which decisions had been made off numbers that were subtly wrong—ran a quarter and burned WAY too much engineering time—$400,000 of it, once someone finally added up the hours—before they had an answer they trusted. The cost of the decisions themselves—the ad budget poured into a campaign that only looked like it was working—was larger and harder to pin down, which is its own kind of expensive.

Of course the hard part here is that ... in large part ... nothing was broken! Their data was clean, the labels were disciplined—they had real ground truth where they needed it. Yet that didn't prevent the system from delivering the wrong answers. The architecture itself was the bug: a pattern that *guaranteed* two implementations of the same logic would eventually drift, because keeping them in sync depended on human memory and good intentions, and those are the two least reliable components in any system.

This chapter is about the patterns that determine whether your clean data yields one trustworthy answer or two warring ones. Architecture is not the glamorous part of this book, but if done wrong, it makes every other chapter's work irrelevant.

---

## 8.1 Warehouses vs Lakes vs Lakehouses

Start with where the data physically sits.

The **data warehouse** is the old money of the data world. Snowflake, BigQuery, Redshift, and their ancestors going back to Teradata. You load structured, cleaned data into a system that provides fast SQL, governance, transactions, and the confidence that the number you queried is the one that exists. The catch is that warehouses charge you for that comfort, by the byte and by the query, and the bill scales with your ambition. The failure mode here starts with something that looks harmless: someone points a dashboard at a warehouse, sets it to auto-refresh every five minutes, and magically you've built a machine that converts idle browser tabs into a five-figure monthly invoice. A single mis-scheduled `SELECT *` on a fact table can do more damage to a budget than an actual outage.

Then came the **data lake**. Dump everything—structured, unstructured, logs, images, the JSON nightmares from Chapter 3—into cheap object storage (S3, GCS, ADLS) in whatever format it arrived, and figure out what it means later. Storage at lake prices is so cheap it's practically free compared to warehouse storage. The problem is that "figure out what it means later" turns out to be the entire job, and most companies never do it. A data lake with no catalog, no schema discipline, and no ownership is not a lake. It's a swamp, and teams can pour years of data in and then act surprised that nobody can find anything, nothing is trustworthy, and the same dataset exists in nine slightly different copies because nobody could confirm which one was real.

The swamp has a signature you can spot from miles away. Someone asks a simple question—"how many active customers did we have in March?"—and three analysts come back with three different numbers, each pulled from a different `customers_final_v2_USE_THIS` directory, each defensible, none authoritative. The cheap storage saved a bunch of money that was promptly wasted by putting eight people in a room for an afternoon, arguing about which number was real instead of acting on any of them. Then comes the slower cost: the erosion of trust that sets in once people learn the data can't be believed. A lake without governance doesn't save you money; it defers the bill and charges interest, which is the exact data-quality-debt dynamic Chapter 4 warned about, now poured into a reservoir.

The **lakehouse** is the synthesis, and Chapter 3 already introduced the table formats that make it work—Iceberg, Hudi, Delta. The pitch is simple and it holds up: keep your data in cheap object storage in open columnar formats, but lay a metadata layer on top that adds the things you actually missed from the warehouse—ACID transactions, schema enforcement, time travel, the ability to update a row without rewriting a terabyte. You get most of what the warehouse gave you without the warehouse's storage bill. This ALSO makes sense, as it's a fair response to a decade of companies bleeding money running a warehouse *and* a lake *and* a brittle pipeline shuttling data between them, paying twice for infrastructure and once more for the engineers who babysat the seam.

The decision is less religious than the vendors would have you believe. If your data is mostly structured, your scale is moderate, and you value a low-drama operational life over a low bill, a warehouse is a perfectly adult choice—don't let anyone shame you into building a lakehouse you don't need. If you're drowning in volume and variety and your warehouse bill has started showing up in finance meetings with your name next to it, the lakehouse pattern is how you stop the bleeding without setting your data on fire. The swamp is never the right choice; it's just the one you arrive at by accident when you pick "lake" and skip the discipline.

---

## 8.2 Lambda vs Kappa: The Two-Codebases Tax

The e-commerce company at the top of this chapter was running a **Lambda architecture**, which is one of the most common patterns in production and one of the most dangerous, in a way that never announces itself.

Lambda, in the original sense Nathan Marz described, means you run two paths in parallel. A **batch layer** processes all your data, slowly but completely and correctly, producing authoritative results. A **speed layer** processes the recent data fast but approximately, so you have something to look at right now. A serving layer merges the two: fast-but-rough for the present, slow-but-right for the past. On paper, it can look like the best of both worlds. You get both low latency and eventual correctness, and each layer does what it's good at.

In practice, Lambda's fatal flaw is the $400,000 one: **you have written your business logic twice.** Once in the batch system, once in the streaming system, in two different frameworks with two different execution models, and Murphy's law will spend the rest of your company's life pulling those two implementations apart. Every feature, every bug fix, every regulatory change now has to land in two places, verified two ways, by people who may not even know the other place exists. Keeping them in sync is the thing you have to actively fight for, every single sprint, forever.

It looks innocent at the code level, which is exactly why it survives review. The whole disaster, in miniature—one metric, two correct implementations of "orders in the last hour":

```python
# Streaming path (Flink): event time, sliding window, fixed at the boundary bug
count = orders.key_by(region).window(Sliding(hours=1)).count()
# the fix: window is now (now - 3600s, now], exclusive on the left

# Batch path (Spark): processing date, the boundary bug never fixed
count = (df.filter(df.ts >= window_start)   # inclusive on BOTH ends
           .filter(df.ts <= window_end)
           .groupBy("region").count())
```

Neither file is wrong on its own, and both pass their own tests. But one counts an order sitting exactly on the boundary and the other counts it twice, and that one-row difference, multiplied across millions of orders and rolled up into the headline revenue number, is enough to make a flat day look like growth or turn a real record into a shrug. No single line of code looks like a $400,000 bug. The bug lives in the *architecture*—in the decision to have two files at all.

**Kappa architecture**, as described by Jay Kreps, makes a different bet: delete the batch layer, and treat *everything* as a stream. Your historical reprocessing isn't a separate batch codebase—it's just replaying the same stream through the same streaming code from an earlier offset. This gives you one implementation of your logic, and one place to fix a bug. If you need to recompute history because you changed the logic, you rewind the log and run the same code over the old events. The number can only diverge from itself, which is to say it can't.

There is no free lunch; Kappa requires additional architecture. It demands a durable, replayable log (this is the world Kafka was built for), it asks more of your streaming infrastructure, and "just replay everything" is a sentence that gets a lot heavier when "everything" is petabytes. But the architectural principle underneath it is the one to carry out of this section, even if you never go full Kappa: **every time you implement the same business logic twice, you have created a reconciliation project with a delayed fuse.** Sometimes two implementations are genuinely necessary. Just know that you've signed up for the tax, and budget for the audit that proves they still agree.

---

## 8.3 Column-Oriented Storage at the Architecture Level

Chapter 3 covered *why* columnar formats win—Parquet's compression and predicate pushdown, Arrow's zero-copy interchange. What matters here is what columnar storage means when you zoom out from a single file to an entire architecture.

Two things. First, **columnar is what makes the lakehouse economically possible at all.** The reason you can run warehouse-grade analytics directly against files in object storage is that the engine only reads the columns and row groups a query touches. Query three columns out of two hundred, and you've read one and a half percent of the bytes. Without that, "SQL on a data lake" would be a polite way of saying "scan a terabyte to answer every question," and the lakehouse would collapse under its own bill.

Second, **Arrow has become the lingua franca between the boxes in your architecture diagram.** The expensive, invisible tax in any multi-tool data stack used to be serialization—every hop between Spark and pandas and your model and your query engine meant data putting its coat on and taking it off again, as Chapter 3 put it. When the components all speak Arrow, the data moves between them without that conversion. At the scale of a single notebook, that's a nice speedup. At the scale of an architecture where data crosses five systems on its way to a model, it's the difference between a pipeline that's mostly working and a pipeline that's mostly copying bytes. When you're drawing the diagram, the question to ask at every arrow between two boxes is: *what format does the data take crossing this line, and how many times are we paying to reshape it?* The arrows are where architectures bleed performance.

There's a third consequence that matters specifically for the readers of this book, the ones whose pipelines end at a model. The handoff from your data layer to your training code is the most violent reshape in most architectures—the place where a nice columnar dataset gets exploded into the row-by-row, tensor-shaped thing a model wants, often through pandas, often three times, often in a notebook nobody profiles. A columnar-all-the-way-down architecture pushes that boundary as late as possible: Arrow into the feature pipeline, Arrow into the framework's data loader, and the reshape happens once, at the last possible moment, instead of at every hop. It's the least glamorous performance win available, and on real training pipelines it routinely reclaims more wall-clock time than swapping the model onto a bigger GPU.

---

## 8.4 Cloud-Native Platforms: AWS, GCP, Azure

You will run this somewhere, and for many, many folks, that somewhere is one of three clouds. I'm not going to pretend there's a clean winner, because there isn't, and anyone who tells you their cloud is universally best is selling something or has only ever used one.

**AWS** is the widest and messiest. S3 is the de facto floor of the entire data world—half the tools in this book assume it exists—and the surrounding cast (Redshift, Athena, Glue, EMR, Kinesis) covers every pattern you could want. The cost of that breadth is that AWS gives you fifteen ways to do everything and zero opinions about which one you should pick. You get maximum flexibility and maximum rope. Teams that thrive on AWS have the discipline to choose; teams that don't end up with all fifteen services running at once and a bill nobody can explain.

**GCP** has the strongest opinion, and it's mostly good. BigQuery is, no exaggeration, one of the best things any cloud has shipped for analytics—serverless, fast, and structured so that the easy path is also the scalable one. If your center of gravity is "we want to do analytics and ML on big data without becoming infrastructure plumbers," GCP's integrated story is hard to beat. The trade-off is a smaller ecosystem and the recurring corporate anxiety about Google's enthusiasm for sunsetting things.

**Azure** wins on a single, frequently decisive point: you already have it. If your company runs on Microsoft (Active Directory, Office, enterprise agreements, a procurement department that already trusts Redmond) then Synapse, Fabric, and ADLS arrive pre-blessed, and the integration with the identity and tooling your company already uses removes a category of friction that, in a regulated enterprise, can matter more than any benchmark. This is the federal and Fortune-500 reality: the best platform is very often the one that legal and procurement have already approved.

There's a quieter form of lock-in than file formats: **data gravity.** Chapter 4 walked through egress fees—the racket where cloud vendors let you bring data in for free and charge you to take it out. That asymmetry is not an accident. Once you have a few petabytes sitting in one cloud, the cost and time to move them elsewhere become their own deterrent, regardless of format. Your data develops gravity, and gravity makes the migration you keep meaning to do quietly impossible. The practical defense is to know your egress number before it's load-bearing, and to resist the pattern—covered in Chapter 4—where data ping-pongs across regions and clouds, racking up transfer fees nobody put in the budget. Architecture is partly the art of not letting your own data take you hostage.

The differences between these three at the feature level are real but narrowing every year, and they are almost never what sinks you. **What sinks you is the lock-in you chose without noticing.** The moment your data only exists in one vendor's proprietary format, queryable only by that vendor's engine, you have handed them your pricing leverage forever. This is the deeper reason the open table formats from Chapter 3 matter so much: keeping your data in Parquet and Iceberg on object storage means the storage layer is portable even when the compute isn't. Pick a cloud for its strengths, absolutely—but keep your data in formats you could walk out the door with, because someday you may want to.

---

## 8.5 What the Giants Actually Did

The companies everyone cites—Netflix, Uber, Airbnb—are cited for a reason, but the reason is usually misunderstood. People look at them and think the lesson is "use what they use." It isn't. Most of you are not operating at that scale, and cargo-culting a streaming giant's architecture onto a problem that fits in Postgres is its own disaster. The real lesson is in *what each of them was forced to build*, because what you build under extreme pressure reveals what the existing tools couldn't do.

**Netflix** hit the limits of treating a data lake like a real database and, rather than retreat to a warehouse, built the missing piece: Apache Iceberg, the open table format that brings ACID transactions, schema evolution, and correctness to files in object storage. The pattern worth stealing isn't "be like Netflix." It's the conviction behind Iceberg: separate your storage from your compute, keep storage open, and add database guarantees as a layer rather than buying them bundled in a proprietary box. That's the lakehouse thesis, paid for in production by someone who had no choice.

**Uber** had a different pain. They needed to update and correct records in a massive lake—late-arriving data, trip corrections, GDPR deletions—and the lake's "write once, never change" assumption made that brutal. So they built **Apache Hudi**, which brought efficient upserts and incremental processing to lake storage: change the rows that changed instead of rewriting the dataset. The transferable lesson is to notice when your access pattern (frequent updates) is at odds with your storage's assumption (immutability), because that fight is a tax you pay on every single write until you fix the architecture, not the code.

**Airbnb** built **Apache Airflow** because their problem wasn't storage at all—it was orchestration, the dependency hell of hundreds of interlocking data jobs where job C must not run until A and B both succeed, and a failure at 2 a.m. shouldn't silently poison everything downstream. Airflow made those dependencies explicit, scheduled, and observable. The lesson here is the one teams learn last: at a certain scale, *coordinating* your data jobs becomes a harder problem than any individual job, and "we'll just chain some cron jobs" is the opening line of a future incident report.

Every one of these companies kept their data in open formats and built the missing capability as a sharable layer—which is why you've heard of these tools at all. They open-sourced the patterns and, as a result, you can adopt Iceberg, Hudi, and Airflow today without operating at their scale. That is the actual gift.

---

## 8.6 Choosing the Right Architecture for Your Scale

The most expensive architecture mistake is picking the *biggest* tool—building the petabyte-scale, multi-region, streaming-first cathedral for a workload that is, in cold reality, forty gigabytes that change once a day. Not the wrong tool. The biggest one.

Scale honestly. Most data, at most companies, fits on one large machine. A single Postgres instance, or DuckDB over Parquet files (Chapter 6 already showed you DuckDB profiling a terabyte from a laptop), will carry you much further than the conference talks imply, and it will do it with a fraction of the operational misery. Distributed systems don't just cost more in dollars; they cost more in the only currency that's truly scarce, which is your team's attention. Every Kafka cluster, every Spark deployment, every streaming job is a thing that pages someone at night. The 10x cost rule from Chapter 4 has an architectural cousin: complexity you adopt before you need it compounds, because every future change now has to be made inside the cathedral instead of inside the cottage.

A rough ladder:

| Where you actually are | What you probably need | What you're tempted to build |
|---|---|---|
| Gigabytes, daily updates | Postgres, or Parquet + DuckDB | A Spark cluster |
| Low terabytes, mostly analytics | A warehouse (BigQuery/Snowflake) | A lake + streaming |
| Real volume + variety, rising bill | Lakehouse on open formats | A custom in-house platform |
| Truly need sub-second on live events | Add streaming—deliberately | Lambda (now you owe the two-codebase tax) |

The right move is almost always to build for the scale you have plus one step, not the scale you fantasize about at the offsite. You can migrate up the ladder when the pain is real and specific. You cannot easily migrate *down* from a complex system you adopted on spec, because by then a dozen things depend on its existence. Premature scale is just technical debt camouflaged as an architecture diagram.

---

## Quick Wins: Architecture Sanity Checks

**Find your duplicated logic (30 min).** List every metric or feature that's computed in more than one place—batch and streaming, warehouse and app, two teams' pipelines. Each duplicate is a future reconciliation project. You don't have to fix them today; you have to *know they exist*, because right now they're drifting and nobody's watching.

**Audit your most expensive query (20 min).** Open your cloud billing, find the costliest recurring query or job, and ask the brutal question: does anyone read this output? A shocking fraction of cloud spend is dashboards no one opens, refreshing on a schedule no one set on purpose.

**Check your exit (15 min).** Pick your most important dataset and answer honestly: if you had to leave this cloud vendor next quarter, could you take this data with you, or does it only exist in a proprietary format only their engine can read? If it's the latter, that's not a dataset, that's a hostage.

---

## Your Homework

### Exercise 1: Draw the Real Diagram (Time: ~1 hour)

Draw your actual data architecture—not the clean one in the wiki, the real one, with the shadow pipeline someone built in 2022 and the CSV export that three reports secretly depend on. For every arrow between two boxes, label what format the data is in as it crosses. Circle every place the same data exists in two forms. This exercise does not produce a clean diagram, and the mess you find is your actual roadmap.

### Exercise 2: Price Your Architecture (Time: ~45 minutes)

Pull last month's cloud bill and attribute it: storage, compute, egress, idle. Then find the single biggest line item and trace it to a business reason. If you can't explain in one sentence why your largest cost exists, you've found the thing to fix first. (Chapter 4's cost framework is your friend here.)

### Exercise 3: The Two-Number Test (Time: ~30 minutes, possibly alarming)

Pick one important metric. Find every place it's computed. Pull the number from each, for the same time window, and compare. If they disagree, congratulations—you've found a Lambda tax you were already paying and didn't know it. If they agree, write down *how you keep them agreeing*, because that's the discipline you can't afford to lose.

---

## Bridge to Chapter 9

Architecture decides where your data lives and whether it gives you one answer or two. But there's a specific, brutal version of the two-codebases problem that deserves its own chapter: the gap between the features your model trained on and the features it sees in production. Compute a feature one way in your training pipeline and another way in your serving path—the exact Lambda trap transplanted into ML—and your model will rot in production while every offline metric swears it's fine. That problem got bad enough, often enough, that the industry built an entire category of infrastructure to solve it. Chapter 9 is about feature stores: what they are, when you actually need one, and when they're a cathedral you're building for a cottage.

---

*P.S. — The two-number war from the top of this chapter ended the way they always do: not with anyone proving their number was right, but with everyone too exhausted to keep fighting, agreeing to a single source of truth they should have built on day one. Architecture is the set of decisions that are cheap to make at the start and ruinous to change at the end. You will be tempted to defer them, because there's always a feature to ship instead. Defer them anyway and you'll meet them again later, at scale, on fire, with the board watching. Draw the diagram now, while it still fits on one page.*
