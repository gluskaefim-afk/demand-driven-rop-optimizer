An automated inventory model that calibrates the Reorder Point (ROP) for a single SKU by combining Prophet demand forecasting, EOQ calculation, and a day-by-day Monte Carlo / Bootstrap simulation of the inventory system.

### What the model does

1. **Forecast** — a Prophet model is fit on daily demand to estimate total demand over the next 31 days (`yhat`, with `yhat_lower`/`yhat_upper` bounds). This forecast is used only to (a) size the EOQ order quantity and (b) identify historical periods with a comparable demand pattern — it is *not* sampled from directly.
2. **EOQ** — the classic Economic Order Quantity formula sizes the replenishment quantity from forecasted demand, logistics cost per order, and per-unit holding cost.
3. **Period selection for resampling** — historical months whose actual demand falls within the `yhat_lower — yhat_upper` range, and is closest to `yhat` itself, are selected as the resampling pool. This avoids the unrealistic day-level confidence intervals that a pure forecast-residual approach can produce (e.g. negative lower bounds when historical demand includes zeros), while preserving the actual character of demand.
4. **Simulation** — for a given ROP, 1,000 independent 31-day trajectories are simulated. Each day: demand is drawn (bootstrap, with replacement) from the selected historical pool, stock is decremented, holding cost accrues, and a replenishment order is triggered once stock crosses the ROP threshold and arrives after a fixed lead time.
5. **Calibration loop** — ROP is stepped up or down until the simulated worst-case Cycle Service Level (CSL) falls inside a target corridor (95–100%).
### Output metrics
 
The model reports three distinct numbers, each answering a different question:

| Metric | Question it answers |
|---|---|
| **Cycle Service Level (99.9th percentile)** | In a near-worst-case month, what fraction of days is the system stocked out on? |
| **Fill Rate** (aggregated across all 1,000 simulations) | Over long-run operation, what fraction of total demand is served directly from stock? |
| **Average cost** | What is the expected holding + ordering cost per month at this ROP? |
 
### Goal

The model optimizes for **resilience to demand variability**, not average-case cost minimization. This is why the calibration criterion is the 99.9th-percentile CSL (a near-worst-case guarantee) rather than an average-case fill rate — the ROP produced is deliberately conservative, chosen to hold up even in an unusually volatile month.

### Limitations
 
- Optimizes logistics costs only (holding + ordering); does not account for stockout/downtime losses in the objective function.
- Single SKU, single warehouse — no cross-SKU or multi-echelon interactions.
- Lead time is treated as fixed and deterministic, not itself stochastic.
- The historical-period selection step (`yhat_lower — yhat_upper` filtering) is currently precomputed and hardcoded as a fixed list of months in this notebook rather than recomputed inline — full reproducibility requires re-running that filtering step if the underlying dataset changes.
- Dataset used for demonstration is synthetically generated, not production data.
- 
### Stack

`pandas`, `numpy`, `prophet`, `matplotlib`
