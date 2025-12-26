Cloud egress fees are where dreams go to die. But the dollar cost isn't even the real problem. It's the latency tax on every decision your system makes.

Standard pattern I see:

1. Store data in us-east-1 (cheapest!)
2. Run ML training in us-west-2 (GPUs available!)
3. Serve predictions from eu-west-1 (users are there!)

Every byte crossing regions costs money. AWS charges $0.02/GB for cross-region transfer. A company moving 500 GB daily for model training, syncing feature stores, and aggregating logs might burn $1,800/month in bandwidth nobody budgeted for.

But here's what doesn't show up on any invoice: the 70-150ms latency penalty every time data crosses regions. When your fraud detection model needs fresh features from a store 3,000 miles away, that roundtrip isn't free. When your recommendation engine waits for user signals to traverse the Atlantic, that's not "network overhead." That's degraded predictions. When your real-time bidding system loses auctions because feature retrieval adds 100ms, that's revenue evaporating.

The bandwidth costs are a rounding error compared to the decisions you're making with stale data. Or worse, the decisions you're not making at all because the latency budget doesn't allow for the enrichment that would actually matter.

This is why data locality isn't a performance optimization. It's an architectural requirement. The question isn't "how do we afford to move this data?" It's "why are we moving this data at all?" Every cross-region transfer is a confession that you put your compute in the wrong place. The cheapest byte to transfer is the one that never leaves the region where it was created.
