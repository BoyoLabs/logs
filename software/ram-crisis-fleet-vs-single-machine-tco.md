# Old-Hardware Fleet vs. One Modern Machine: A TCO Simulation During the 2026 RAM Crisis
**Date:** 2026-08-06

## Context & Intent

Starting question: with 2026's DRAM shortage driving new-RAM prices up 4-5x
(Samsung/SK Hynix/Micron shifting fab capacity to HBM for AI datacenters,
since HBM and consumer DDR5 come off the same lines), does it become
genuinely more cost-effective to reach a given amount of RAM/compute by
buying a fleet of old, cheap secondhand machines (OptiPlex-class SFF
desktops, old laptops) instead of one modern "beefy" machine? And if so,
by how much, and does that hold up over time or just at the moment of
purchase?

This was explicitly scoped as a rigorous, numbers-first simulation rather
than a built tool — no existing generic calculator fits this specific
comparison (cloud-vs-on-prem TCO calculators exist, this scenario doesn't
map onto them), so the answer is a script plus its actual output, not a
gut-feel estimate.

## The Execution

**Inputs, each labeled by how they were obtained:**

- **SOURCED** — Dell OptiPlex 7020 SFF (i5-6500, 8GB RAM, no storage):
  real eBay listing, $22.50, checked 2026-08-06.
- **ESTIMATED** — a cheap SSD to make that unit usable: $15.
- **SOURCED** — PassMark multi-thread score, Intel Core i5-6500: 5,601.
- **ESTIMATED** — idle/load power draw for this class of SFF desktop:
  28W idle / 55W load (published figures for mid-2010s SFF office
  desktops commonly fall in the 20-35W idle range; 28W is a representative
  midpoint, not a measurement of this specific unit).
- **ESTIMATED** — barebones modern platform (Ryzen 5 8600G + motherboard +
  PSU + case + cooler + 1TB NVMe, RAM excluded): $515 at current retail.
- **SOURCED** — PassMark multi-thread score, AMD Ryzen 5 8600G: 25,197.
- **SOURCED** — current DDR5 pricing during the shortage: ~$12-14/GB;
  real 128GB (4x32GB) kit listings found the same day ranged $1,600-$2,366
  (~$12.50-$18.50/GB depending on brand/speed). $13/GB used as a clean,
  slightly-conservative midpoint.
- **ESTIMATED** — modern-desktop idle/load power: 35W idle / 90W load.
- **ESTIMATED** — electricity rate: $0.12 / $0.16 / $0.22 per kWh
  (low/mid/high bracket spanning a typical US residential range), and two
  usage patterns (10% loaded / 90% idle vs. 35% loaded / 65% idle), both
  running 24/7.

**Method:** for target aggregate RAM of 64GB, 128GB, and 256GB, compute
CapEx (hardware cost) and OpEx (electricity cost over time, at each
rate/usage combination) for both options, then solve for the crossover
point — the time, in years, at which the modern machine's total cost of
ownership drops below the fleet's, if it ever does within a 10-year cap.

Full simulation script:

```python
#!/usr/bin/env python3
import math

FLEET_UNIT_BASE_COST = 22.50       # SOURCED (eBay, 2026-08-06)
FLEET_UNIT_RAM_GB = 8
FLEET_UNIT_STORAGE_COST = 15.00    # ESTIMATED
FLEET_UNIT_COST = FLEET_UNIT_BASE_COST + FLEET_UNIT_STORAGE_COST
FLEET_UNIT_PASSMARK = 5601         # SOURCED
FLEET_UNIT_IDLE_WATTS = 28         # ESTIMATED
FLEET_UNIT_LOAD_WATTS = 55         # ESTIMATED

BEEFY_BAREBONES_COST = 515.00      # ESTIMATED
BEEFY_PASSMARK = 25197             # SOURCED
BEEFY_DRAM_PRICE_PER_GB = 13.00    # SOURCED (midpoint of real kit listings)
BEEFY_IDLE_WATTS = 35              # ESTIMATED
BEEFY_LOAD_WATTS = 90              # ESTIMATED

RAM_TARGETS_GB = [64, 128, 256]
ELECTRICITY_RATES = {"low ($0.12/kWh)": 0.12, "mid ($0.16/kWh)": 0.16, "high ($0.22/kWh)": 0.22}
USAGE_PATTERNS = {"idle-mostly (10% loaded)": 0.10, "mixed (35% loaded)": 0.35}
HOURS_PER_YEAR = 24 * 365

def fleet_config(target_gb):
    n = math.ceil(target_gb / FLEET_UNIT_RAM_GB)
    return dict(n_units=n, capex=n * FLEET_UNIT_COST, passmark=n * FLEET_UNIT_PASSMARK,
                idle_w=n * FLEET_UNIT_IDLE_WATTS, load_w=n * FLEET_UNIT_LOAD_WATTS,
                ram_actual=n * FLEET_UNIT_RAM_GB)

def beefy_config(target_gb):
    return dict(n_units=1, capex=BEEFY_BAREBONES_COST + target_gb * BEEFY_DRAM_PRICE_PER_GB,
                passmark=BEEFY_PASSMARK, idle_w=BEEFY_IDLE_WATTS, load_w=BEEFY_LOAD_WATTS,
                ram_actual=target_gb)

def avg_watts(cfg, loaded_fraction):
    return cfg["idle_w"] * (1 - loaded_fraction) + cfg["load_w"] * loaded_fraction

def opex(cfg, loaded_fraction, rate_per_kwh, years):
    w = avg_watts(cfg, loaded_fraction)
    return (w / 1000) * HOURS_PER_YEAR * rate_per_kwh * years

def crossover_year(fleet_cfg, beefy_cfg, loaded_fraction, rate_per_kwh):
    fleet_w, beefy_w = avg_watts(fleet_cfg, loaded_fraction), avg_watts(beefy_cfg, loaded_fraction)
    fleet_slope = (fleet_w / 1000) * HOURS_PER_YEAR * rate_per_kwh
    beefy_slope = (beefy_w / 1000) * HOURS_PER_YEAR * rate_per_kwh
    if beefy_cfg["capex"] <= fleet_cfg["capex"]:
        return 0.0
    if beefy_slope >= fleet_slope:
        return None
    t = (beefy_cfg["capex"] - fleet_cfg["capex"]) / (fleet_slope - beefy_slope)
    return t if t <= 10 else None
```

(Full script including the print/report loop lives with this entry's
working files; the logic above is complete and reproducible on its own.)

## Findings

**CapEx: the fleet wins by 3-4.5x, as expected.** For 128GB aggregate RAM:
16 old units at $600 total vs. one modern machine at $2,179 — a 3.6x gap,
almost entirely explained by DDR5's current price, not by anything about
the CPUs or platforms themselves.

**Compute-per-dollar: the fleet wins too, right now — which contradicts
the naive assumption going in.** The working assumption before running
this was "old CPUs are ~4.5x slower per machine (PassMark 5,601 vs
25,197), so a CPU-bound workload should favor the modern machine's fewer,
faster cores." The simulation shows the opposite at current prices: the
fleet delivers **$0.0067 per PassMark point** vs. the modern machine's
**$0.087 per PassMark point** at 128GB — roughly **13x** better
compute-per-dollar for the fleet. This isn't because the old CPUs got
faster; it's because the modern machine's total cost is now dominated by
RAM, not CPU, so its fixed compute budget is being priced against an
inflated hardware cost that has nothing to do with compute. **The RAM
crisis doesn't just make RAM expensive — it makes the whole machine's
compute-per-dollar look worse, since RAM is now the majority line item.**

**TCO over time: the fleet's advantage erodes fast, and the crossover
arrives sooner than intuition suggests.** Because idle power scales
linearly with fleet size while the modern machine's draw stays flat, the
break-even point where the modern machine becomes cheaper *overall*
lands well inside a normal hardware lifespan:

| Target | Fleet CapEx | Fleet 5yr TCO | Beefy CapEx | Beefy 5yr TCO | Crossover (mixed use, $0.16/kWh) |
|---|---|---|---|---|---|
| 64GB  | $300    | $2,400  | $1,347  | $1,727  | 3.04 yr |
| 128GB | $600    | $4,799  | $2,179  | $2,559  | 2.07 yr |
| 256GB | $1,200  | $9,598  | $3,843  | $4,223  | 1.65 yr |

Across every electricity-rate and usage-pattern combination tested,
crossover ranged from **1.2 to 4.9 years** — never "never," and often much
sooner than a hardware refurb's expected service life. Larger RAM targets
cross over *faster*, not slower, because more RAM means more fleet units,
which means more aggregate idle draw stacking up in parallel.

## Conclusion

There's no single winner — the honest answer is that the right choice
depends on time horizon and how much RAM is actually needed, and the
crossover point is close enough to matter, not a rounding error:

- **Short-lived project, or CapEx is the binding constraint:** the fleet
  wins decisively on every axis (cost, compute-per-dollar) at the moment
  of purchase, current RAM prices make this especially lopsided right now.
- **Always-on service expected to run 2+ years, especially at higher RAM
  targets:** the fleet's power draw compounds against it faster than it
  looks like it should, and a single modern machine is very plausibly
  cheaper by the time you'd actually be relying on the setup long-term.
- **The RAM crisis specifically changes the *shape* of this tradeoff**, not
  just the numbers — normally a modern machine's higher compute-per-dollar
  would be a point in its favor; right now, DDR5 pricing is expensive
  enough to erase that advantage entirely, at least until fab capacity
  meaningfully recovers (most estimates: not before late 2027).

Standing lesson for next time this framing comes up: "cheap old hardware
looks like a clear win" is a CapEx-only intuition. Once power is counted
honestly, the real crossover point is a genuinely useful number to compute
before committing to either side, not an afterthought.

## Prompt to Reproduce This Simulation

The script above is disposable on purpose — it was built against prices
that were only accurate on 2026-08-06, and DRAM pricing in particular has
been moving by double digits in weeks. Paste this to an AI assistant with
web search access to run a genuinely current version rather than trusting
this entry's numbers to still be accurate later:

```
Run a rigorous, numbers-first simulation comparing two ways to reach a target amount of RAM/compute: (A) a fleet of cheap old secondhand desktops/laptops (e.g. Dell OptiPlex-class SFF units), vs (B) one modern "beefy" single machine, sized to the same target RAM.

1. Research CURRENT real prices before doing anything else -- do not reuse any numbers from a prior run of this simulation, RAM and hardware prices move fast (they can move by double digits in weeks during a memory shortage). Specifically look up:
   - A real current listing price for a cheap old SFF desktop or laptop (with RAM, minus storage if needed) -- pick one you can point to, not a guess.
   - Current $/GB pricing for the RAM type a modern build would use (e.g. DDR5), from actual current kit listings, not an assumed number.
   - A current barebones cost for a modern desktop platform (CPU+motherboard+PSU+case+cooler+basic storage), excluding RAM.
   - A CPU benchmark comparison (e.g. PassMark multi-thread) between the old machine's CPU and the modern machine's CPU, so compute-per-dollar can be computed honestly, not assumed.
   - Idle and loaded power draw estimates for both machine classes.
   Label every number SOURCED (from a real listing/benchmark you found) or ESTIMATED (a reasonable figure you couldn't pin to a specific source) -- never blend the two without saying which is which.

2. Write a script that computes, for a few target RAM capacities (e.g. 64GB/128GB/256GB):
   - CapEx for both options.
   - $/GB of RAM and $/benchmark-point of compute for both options.
   - OpEx over time (electricity cost) at a few electricity-rate and usage-pattern assumptions, running 24/7.
   - The TCO crossover point in time -- when (if ever, within ~10 years) the more expensive-upfront option becomes cheaper overall once power is counted.

3. Run it, and report the actual output -- don't estimate the results by hand, run the real computation.

4. Write up the findings honestly, including anything that contradicts the intuitive assumption going in (e.g. if the "obviously more powerful" option doesn't actually win on cost-per-compute given current prices).

5. Once the results are captured in the writeup, delete the script file. It was built against prices that are only accurate as of the day it ran -- leaving it sitting on disk invites someone (including a future you) to re-run it later and mistake stale prices for current ones. The write-up is the durable artifact; the script itself is disposable and re-creatable fresh any time this comparison is worth re-running.
```
