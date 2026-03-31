# Staggered DiD Estimator Explainer

## The shared problem they all solve

Standard **Two-Way Fixed Effects (TWFE)** DiD assumes all treatment effects are the same across groups and time. When games are treated at different times (staggered, as in the 80 cohorts across 2018–2025), TWFE silently uses already-treated games as controls for later-treated games. If treatment effects change over time (which they clearly do — positive at k=0, then negative), this creates **negative weights** that bias estimates. All three methods fix this by only comparing against **not-yet-treated** games.

---

## Local Projections DiD (LP-DiD) — Dube et al. 2023

**How it works:** For each horizon k (e.g., k=3 means "3 months after giveaway"), run a completely separate regression. Each regression compares treated games at that exact horizon against untreated games at the same calendar time.

**Purpose:** Estimate the event-study path in a way that's robust to heterogeneous effects, by never letting early-treated games contaminate late-treated comparisons.

**Pros:**
- Simple to implement and interpret
- Flexible — each period is estimated independently, so no wrong-sign bias
- Good for visualizing the dynamic path

**Cons:**
- Each period uses a different set of not-yet-treated controls, so estimates at different horizons aren't balanced in a symmetric way
- No explicit balancing across cohorts — late cohorts have fewer control games at longer horizons
- Gives a higher k=0 estimate (0.131) than stacked DiD (0.047), suggesting some residual cohort-composition bias

---

## Stacked DiD — Cengiz et al. 2019

**How it works:** For each treatment cohort (e.g., "games given away in Q3 2020"), build a separate mini-dataset ("stack") containing:
1. The treated games from that cohort, with k = -12 to +12
2. All games not yet treated at that cohort's treatment date, with the same relative window

Stack all mini-datasets and run one regression with cohort×game fixed effects and event-time dummies. Games appearing in multiple stacks (controls for early cohorts before their own treatment) are downweighted by 1/n_stacks.

**Purpose:** Create a balanced, apples-to-apples comparison within each cohort. Each treated game is only compared to clean not-yet-treated controls.

**Pros:**
- Most conservative and credible estimate — likely the most conservative estimate of the contemporaneous effect
- Explicitly balanced panels within each cohort
- Avoids both negative weights and cohort-composition bias

**Cons:**
- More complex to build (stacking 80 separate datasets)
- Games appearing in many stacks get heavily downweighted, reducing power
- Requires dropping observations outside the k ∈ [-12, 12] window

---

## Callaway–Sant'Anna (CS) — Callaway & Sant'Anna 2021

**How it works:** Directly estimate **cohort-specific ATTs**: ATT(g, k) = the average treatment effect for cohort g (e.g., "games first given away in Q2 2021") at event time k. Each ATT(g,k) uses only not-yet-treated games as controls for that specific cohort. Then aggregate across cohorts weighted by cohort size.

**Purpose:** Fully transparent about treatment effect heterogeneity. Instead of assuming one average effect, first asks "what's the effect for each batch of games?" then averages.

**Pros:**
- Most principled theoretically — cleanest separation of cohort-specific effects
- Allows inspection of whether early-cohort vs. late-cohort games respond differently
- Aggregation weights are explicit and researcher-controlled

**Cons:**
- Smallest effective sample per cohort — if a cohort is small, that ATT is noisy
- With 80 cohorts, many are tiny, making cohort-level estimates unreliable
- Best used as a robustness check rather than primary estimator in this setting

---

## Summary

| | LP-DiD | Stacked DiD | Callaway–Sant'Anna |
|---|---|---|---|
| **Core idea** | Separate regression per horizon | Stack cohort-specific panels | Cohort-specific ATTs, then aggregate |
| **Controls used** | Not-yet-treated at each period | Not-yet-treated within each stack | Not-yet-treated within each cohort |
| **Balancing** | Implicit | Explicit (within stack) | Explicit (within cohort) |
| **k=0 estimate** | 0.131 (highest) | 0.047 (most conservative) | Similar to TWFE |
| **Best for** | Quick, intuitive event study | Credible, balanced panel | Inspecting cohort heterogeneity |
| **Main weakness** | Subtle cohort-composition bias | Complex build, downweights multi-stack games | Noisy with many small cohorts |

All three estimators agree on the qualitative two-phase story (positive spike at k=0, reversal by k=2, sustained decline through k=12), which is why they function as robustness checks on each other rather than contradicting the TWFE baseline.
