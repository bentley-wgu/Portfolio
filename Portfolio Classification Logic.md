# Portfolio Classification Logic

This document explains how every account is classified into one of four portfolios (AP, GP, MP, WP), how the EP roll-up is derived, and how the FY27 target is calculated. It is written so someone unfamiliar with the calculator can replicate the logic in Excel, Python, SQL, or any other tool.

## Inputs per account

Every account has four pieces of data:

1. **Partner Type.** One of: Corporate, Healthcare, K12 Partner, Transfer Partner, Military.
2. **FY25 System Starts.** Prior fiscal year volume.
3. **FY26 System Starts.** Current fiscal year volume.
4. **Velocity.** Year-over-year growth rate, expressed as a decimal (0.10 = 10%, 1.50 = 150%, etc.). Use the source-of-truth value when available. If it is missing, calculate it from FY25 and FY26 (see "Velocity formula" below).

## Settings per partner type

Each partner type has its own thresholds and a separate growth rate for each of the four portfolios. Defaults:

| Partner Type | Volume Trigger | Velocity Trigger | AP Growth | GP Growth | MP Growth | WP Growth |
|---|---|---|---|---|---|---|
| Corporate | 40 | 1% | 10% | 10% | 10% | 10% |
| Healthcare | 40 | 8% | 10% | 10% | 10% | 10% |
| K12 Partner | 20 | 5% | 10% | 10% | 10% | 10% |
| Transfer Partner | 100 | 8% | 10% | 10% | 10% | 10% |
| Military | 100 | 0% | 10% | 10% | 10% | 10% |

These are the starting points. The calculator allows them to be adjusted per session.

## Velocity formula

Velocity is the year-over-year growth rate from FY25 to FY26.

```
Velocity = (FY26 - FY25) / FY25
```

Special cases:

- If FY25 is 0 and FY26 is greater than 0, treat velocity as infinite (the account is "new").
- If both FY25 and FY26 are 0, velocity is 0.
- A pre-calculated velocity from the source data should be used in place of this formula whenever it is provided, because the source of truth may use unrounded internal values.

## The four portfolios

Every account falls into exactly one portfolio based on two yes/no questions:

1. Is its FY26 volume **above** the partner type's Volume Trigger?
2. Is its velocity **above** the partner type's Velocity Trigger?

| | High Velocity (above trigger) | Low Velocity (at or below trigger) |
|---|---|---|
| **High Volume** (above trigger) | **AP** Anchor Portfolio | **MP** Mature Portfolio |
| **Low Volume** (at or below trigger) | **GP** Growth Portfolio | **WP** Watching Portfolio |

In words:

- **AP (Anchor Portfolio).** Big and growing. These accounts are above the volume threshold and growing faster than the velocity threshold. They anchor the book of business.
- **GP (Growth Portfolio).** Small but growing fast. Below the volume threshold but exceeding the velocity threshold.
- **MP (Mature Portfolio).** Big but flat or shrinking. Above the volume threshold but at or below the velocity threshold.
- **WP (Watching Portfolio).** Small and flat or shrinking. Below the volume threshold and at or below the velocity threshold.

## The EP roll-up

**EP (Executive Portfolio)** is not a separate quadrant. It is the union of AP + GP + MP, in other words every account that is **not** in WP. Use it when reporting on the active book of business, where WP accounts are intentionally excluded.

```
EP accounts = AP accounts + GP accounts + MP accounts
EP FY27 target = sum of FY27 targets across AP, GP, MP
EP FY26 actuals = sum of FY26 actuals across AP, GP, MP
```

## New account override

If an account has FY25 = 0 and FY26 > 0, it goes to **GP** automatically, regardless of how big its FY26 volume is. New accounts are always Growth, never Anchor.

## FY27 target formula

Each account uses the growth rate of the portfolio it landed in.

```
FY27 Target = round( FY26 * (1 + Growth Rate of that portfolio) )
```

Round to the nearest whole number for display. The four growth rates (AP, GP, MP, WP) are configured per partner type and all default to 10%.

## EP blended growth rate

The Avg Growth shown in the EP banner is the volume-weighted blended rate across AP + GP + MP. It is **not** a simple average of the three growth-rate inputs. It uses each portfolio's FY26 volume as the weight:

```
EP FY27 = sum of FY27 targets across AP, GP, MP
EP FY26 = sum of FY26 actuals across AP, GP, MP
EP Avg Growth = (EP FY27 / EP FY26) - 1
```

Using volume weighting means a small AP and a large MP do not pull the blended rate equally. The portfolio with more FY26 volume contributes more to the blended number.

## Worked examples

Using Corporate defaults: Volume Trigger = 40, Velocity Trigger = 1%, Growth Rate = 10%.

**Example 1.** Account with FY25 = 100, FY26 = 120.
- Velocity = (120 - 100) / 100 = 0.20 = 20%.
- FY26 (120) > 40, so High Volume. Velocity (20%) > 1%, so High Velocity.
- Portfolio: **AP** (rolls up into EP).
- FY27 Target = round(120 * 1.10) = 132.

**Example 2.** Account with FY25 = 25, FY26 = 30.
- Velocity = (30 - 25) / 25 = 0.20 = 20%.
- FY26 (30) is not > 40, so Low Volume. Velocity (20%) > 1%, so High Velocity.
- Portfolio: **GP** (rolls up into EP).
- FY27 Target = round(30 * 1.10) = 33.

**Example 3.** Account with FY25 = 200, FY26 = 200.
- Velocity = 0%.
- FY26 (200) > 40, so High Volume. Velocity (0%) is not > 1%, so Low Velocity.
- Portfolio: **MP** (rolls up into EP).
- FY27 Target = round(200 * 1.10) = 220.

**Example 4.** Account with FY25 = 10, FY26 = 8.
- Velocity = (8 - 10) / 10 = -0.20 = -20%.
- FY26 (8) is not > 40, so Low Volume. Velocity (-20%) is not > 1%, so Low Velocity.
- Portfolio: **WP** (excluded from EP).
- FY27 Target = round(8 * 1.10) = 9.

**Example 5.** New account with FY25 = 0, FY26 = 75.
- New account override applies.
- Portfolio: **GP** (even though FY26 of 75 would otherwise be High Volume). Rolls up into EP.
- FY27 Target = round(75 * 1.10) = 83.

## Excel implementation

If `B` = FY26, `C` = FY25, `D` = velocity (decimal), `E` = Volume Trigger, `F` = Velocity Trigger, and `G_AP`, `G_GP`, `G_MP`, `G_WP` are the four growth rates:

```
Portfolio = IF( AND(C=0, B>0), "GP",
              IF( AND(B>E, D>F), "AP",
              IF( AND(B<=E, D>F), "GP",
              IF( AND(B>E, D<=F), "MP", "WP"))))

GrowthRate = CHOOSE( MATCH(Portfolio, {"AP","GP","MP","WP"}, 0), G_AP, G_GP, G_MP, G_WP )

In_EP = IF(Portfolio="WP", 0, 1)

FY27 Target = ROUND(B * (1 + GrowthRate), 0)

EP_FY26   = SUMIFS(B_range, Portfolio_range, "<>WP")
EP_FY27   = SUMIFS(FY27_range, Portfolio_range, "<>WP")
EP_Growth = EP_FY27 / EP_FY26 - 1
```

## Python implementation

```python
def classify(fy25, fy26, velocity, vol_trigger, vel_trigger):
    if fy25 == 0 and fy26 > 0:
        return "GP"
    high_volume = fy26 > vol_trigger
    high_velocity = velocity > vel_trigger
    if high_volume and high_velocity:
        return "AP"
    if not high_volume and high_velocity:
        return "GP"
    if high_volume and not high_velocity:
        return "MP"
    return "WP"

def fy27_target(fy26, growth_rate_for_portfolio):
    return round(fy26 * (1 + growth_rate_for_portfolio))

def in_ep(portfolio):
    return portfolio != "WP"

# Example use with per-portfolio growth rates dict:
growth = {"AP": 0.20, "GP": 0.15, "MP": 0.05, "WP": 0.00}
# For each account:
#   port = classify(...)
#   target = fy27_target(account.fy26, growth[port])
# EP blended growth:
#   ep_fy26 = sum(a.fy26 for a in accounts if a.port != "WP")
#   ep_fy27 = sum(a.target for a in accounts if a.port != "WP")
#   ep_growth = ep_fy27 / ep_fy26 - 1
```

## Notes for replication

- The comparison operator for the triggers is strict greater-than (`>`), not greater-than-or-equal (`>=`). An account exactly at the trigger value is classified as Low (Volume or Velocity).
- Velocity should be expressed as a decimal in the formula. A 5% velocity trigger is 0.05, not 5.
- The growth rate applies uniformly to every account in a partner type. The calculator does not vary the growth rate by portfolio.
- The four quadrants (AP, GP, MP, WP) are mutually exclusive and exhaustive. Every account lands in exactly one of them.
- EP is a derived roll-up, not a quadrant. An account counted in EP is also counted in exactly one of AP, GP, or MP.
