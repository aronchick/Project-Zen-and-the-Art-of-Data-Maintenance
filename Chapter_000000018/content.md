# Chapter 18: Time-Series and Streaming Data

## Or: The Fourth Dimension of Pain

**A high-frequency trading firm had a model predicting price movements. It was right 51% of the time, which in HFT is basically printing money. One day, profits disappeared. The model was still 51% accurate, but they were losing millions.**

The investigation revealed: the network team upgraded fiber optic cables (faster is better, right?). Latency dropped by 3ms - celebration! But the predictions now arrived 3ms earlier than before. The predictions were for the time they *used* to arrive. They were now predicting the past. The past had already happened. You can't trade on yesterday's weather forecast.

**The Fix**: Add 3ms delay back into the system.
**Cost of debugging**: $50 million in losses.
**Lesson**: Time is relative, but your bank account isn't.

---

## 18.1 The Three Types of Time (Yes, Three)

Everyone thinks there's one timestamp. Everyone is wrong. What actually exists in your data:

**Event Time**: When something actually happened in the real world
- Example: Customer clicked "Buy" at 10:30:00 AM

**Ingestion Time**: When your system first saw the data
- Example: Event arrived at your Kafka cluster at 10:31:45 AM

**Processing Time**: When you actually processed the data
- Example: Your batch job ran at 10:35:00 AM

**Which one is your model using?** Spoiler: It's using the wrong one.

I've seen teams debug for WEEKS only to discover their model was training on file creation time, not event time. The model had literally learned that things happen alphabetically by filename. Monday's data came before Tuesday's because "monday.csv" sorts before "tuesday.csv".

---

## 18.2 Window Functions: When Aggregation Beats Raw Events

Many real-world scenarios require aggregated data rather than individual events. Raw event streams can actually mislead your models when normal variance gets misinterpreted as meaningful signals.

A way to approach this situation is through a technique called "Windowing". There are three methods of windowing:

* Tumbling windows
* Sliding windows
* Session windows

### 18.2.1 Tumbling Windows (Discrete Chunks)

Think of tumbling windows like hourly shifts at a factory. When the clock strikes the hour, the current shift ends, counts are tallied, and a fresh shift begins. No event spans multiple windows.

- **Structure**: [0-5min] [5-10min] [10-15min]
- **How it works**: Events from 0:00-4:59 go in window 1. Events from 5:00-9:59 go in window 2. Clean boundaries, no overlap.
- **Advantage**: Simple to implement, no event counted twice, perfect for accounting
- **Problem**: Events at 4:59 and 5:01 are only 2 seconds apart but land in completely different windows, destroying continuity
- **Real-world use**:
  - E-commerce: "Orders per hour" for staffing decisions
  - Banking: "Transactions per day" for reconciliation
  - Manufacturing: "Units produced per shift"
- **The gotcha**: A spike at 4:59 looks unrelated to a spike at 5:01, even though they're the same event

### 18.2.2 Sliding Windows (Overlapping)

Sliding windows are like a security camera that takes a photo every minute but captures the last 5 minutes in each shot. You get smooth transitions because each window overlaps with its neighbors.

- **Structure**: [0-5min], [1-6min], [2-7min], [3-8min]...
- **How it works**: Every minute (or second, or whatever), you create a new 5-minute window. An event at 3:30 appears in windows starting at 0:00, 1:00, 2:00, and 3:00.
- **Advantage**: Smooth transitions, no artificial boundaries, patterns visible across window edges
- **Problem**: 5x the computation (for minute-by-minute slides), events counted in multiple windows, storage explosion
- **Real-world use**:
  - Network monitoring: "5-minute average CPU usage" updating every minute
  - Stock trading: "20-day moving average" recalculated daily
  - Website analytics: "Rolling 24-hour unique visitors"
- **The gotcha**: That spike in traffic? It'll affect 5 different windows, making it look 5x bigger than it was

### 18.2.3 Session Windows (Activity-Based)

Session windows don't care about the clock. They care about user behavior. A session starts when activity begins and ends after a period of inactivity. Like conversations—they last as long as people keep talking.

- **Structure**: [user active 9:00-9:15]...(30 min gap)...[user active 9:45-10:30]
- **How it works**: Define an inactivity threshold (say, 30 minutes). Any gap longer than that ends the current session. Sessions can be 5 minutes or 5 hours.
- **Advantage**: Matches actual user behavior, captures complete interactions, doesn't split user journeys
- **Problem**:
  - Every user has different patterns
  - Sessions have wildly different lengths
  - Can't pre-allocate resources (you don't know when they'll end)
  - Debugging is hell ("Why is this session 14 hours long?")
- **Real-world use**:
  - Google Analytics: User sessions on websites
  - Music streaming: Listening sessions for recommendations
  - Gaming: Play sessions for engagement metrics
  - IDE usage: Coding sessions for productivity analysis
- **The gotcha**: One user who leaves their tab open ruins all your averages. You'll need timeout logic, maximum session lengths, and lots of edge case handling.

### Quick Decision Guide

- **Need exact counts?** → Tumbling windows
- **Need smooth metrics?** → Sliding windows
- **Need to understand user behavior?** → Session windows
- **Need real-time alerts?** → Sliding windows with short slides
- **Need billing/accounting?** → Tumbling windows (no double-counting)
- **Need to minimize compute?** → Tumbling windows
- **Don't know what you need?** → Start with tumbling, it's simplest

---

## 18.3 Late-Arriving Data: The Bane of Stream Processing

[TODO: Expand section on watermarks, allowed lateness, and handling out-of-order events]

In real systems, data doesn't arrive in order. Mobile apps buffer events during subway rides. Edge devices batch uploads. Networks have variable latency. Your "real-time" pipeline needs to handle data that's seconds, minutes, or hours late.

The questions you need to answer:

1. **How late is too late?** At some point, you have to close the window and emit results.
2. **What do you do with late data?** Ignore it? Update previous results? Create correction events?
3. **How do you track "lateness"?** You need watermarks - estimates of how complete your data is.

---

## 18.4 Time Zones: The Universal Footgun

[TODO: Add comprehensive section on timezone handling disasters]

Quick rules that will save your sanity:

1. **Store everything in UTC.** Always. No exceptions.
2. **Convert to local time only at display time.** Never in your pipeline.
3. **Be explicit about what timezone you're using.** "2024-01-15 10:30:00" is meaningless without a timezone.
4. **Remember DST exists.** 2:30 AM might happen twice, or not at all.
5. **Test with Australian data.** Their DST is opposite and will break your assumptions.

---

## 18.5 Temporal Leakage: The Silent Accuracy Killer

[TODO: Expand section on preventing future data from leaking into training]

The most insidious bug in time-series ML: accidentally using future information to predict the past.

Common sources of leakage:

- **Shuffling before splitting**: Train/test split must be temporal, not random
- **Feature engineering on full dataset**: Calculate statistics only on training period
- **Target encoding with future data**: Labels computed after the prediction point
- **Joining tables without time awareness**: Latest customer info joined to old transactions

---

## 18.6 Seasonality and Stationarity

[TODO: Add content on detecting and handling seasonal patterns, trend removal, differencing]

---

## 18.7 Resampling and Interpolation

[TODO: Add content on upsampling, downsampling, handling irregular time series]

---

## 18.8 Streaming Architecture Patterns

[TODO: Add content on Lambda vs Kappa architecture, exactly-once processing, state management]

---

## 18.9 Case Studies

### The $50 Million Dollar Millisecond (Expanded)

[Full case study from opening - expand with technical details on how to prevent similar issues]

### The Seasonality That Wasn't

[TODO: Add case study on mistaking data collection artifacts for real patterns]

### The Timezone Apocalypse

[TODO: Add case study on DST-related failures]

---

## 18.10 Practical Toolkit

### Time-Series Audit Script

```python
def audit_timestamps(df, time_col):
    """Check if your timestamps make sense"""
    df = df.sort_values(time_col)
    df['time_diff'] = df[time_col].diff()
    
    print("Time audit results:")
    print(f"Negative time jumps: {(df['time_diff'] < pd.Timedelta(0)).sum()}")
    print(f"Suspiciously large gaps: {(df['time_diff'] > df['time_diff'].mean() + 3*df['time_diff'].std()).sum()}")
    print(f"Duplicate timestamps: {df[time_col].duplicated().sum()}")
    print(f"Weekend data points: {df[df[time_col].dt.dayofweek.isin([5,6])].shape[0]}")
    
    # Check for timezone issues
    if df[time_col].dt.tz is None:
        print("WARNING: Timestamps have no timezone info!")
    
    # Check for suspicious patterns
    hour_dist = df[time_col].dt.hour.value_counts()
    if hour_dist.std() > hour_dist.mean():
        print("WARNING: Uneven hourly distribution - possible collection bias")
    
    if any([
        (df['time_diff'] < pd.Timedelta(0)).sum() > 0,
        df[time_col].duplicated().sum() > 0
    ]):
        print("\nYou have time problems. Fix them before modeling.")
```

### Temporal Train/Test Split

```python
def temporal_split(df, time_col, train_end, test_start=None):
    """
    Split data temporally, not randomly.
    
    Args:
        df: DataFrame with time column
        time_col: Name of timestamp column
        train_end: Last timestamp for training data
        test_start: First timestamp for test data (default: train_end)
    
    Returns:
        train_df, test_df
    """
    if test_start is None:
        test_start = train_end
    
    train = df[df[time_col] <= train_end].copy()
    test = df[df[time_col] >= test_start].copy()
    
    # Sanity checks
    assert train[time_col].max() <= test[time_col].min(), \
        "Temporal leakage detected! Train and test periods overlap."
    
    print(f"Train: {len(train)} rows, {train[time_col].min()} to {train[time_col].max()}")
    print(f"Test: {len(test)} rows, {test[time_col].min()} to {test[time_col].max()}")
    
    return train, test
```

---

**Visual Note Summary for Chapter 18:**
1. Three types of time diagram (event/ingestion/processing)
2. Window function comparison (tumbling/sliding/session)
3. Late data handling flowchart
4. Temporal leakage detection checklist
5. Timezone conversion reference chart

---

**Code Repository Note**: All code examples from this chapter are available at `https://github.com/aronchick/zen-and-the-art-of-data-maintenance/tree/main/Chapter_000000018`
