# DDR5 SO-DIMM-to-DIMM Adapter Experiment on AM5 — 2026-08-17

**Status: 4-DIMM mix tested, failed to POST, reverted to native config.** Adapters
arrived 2026-08-19 (one day later than the original 08-18 estimate). This entry was
started before the test itself, as a running log — Context & Intent was written first;
Execution and Findings below were filled in after the actual 4-DIMM boot test ran, not
backfilled to look predictive. The matched-pair fallback test (2 SO-DIMM+adapter alone,
no native UDIMM) described below has not been run yet — that's the next step if this
gets picked back up.

## Context & Intent

The board (AM5, current native memory: 2x8GB Team Group UD5-6000 UDIMM, running at
stock 4800 MT/s) has 2 empty DIMM slots. Two 8GB DDR5 SO-DIMMs (laptop-form-factor
memory) were already on hand at no cost, left over from elsewhere. Two SO-DIMM-to-DIMM
adapter cards — the kind that lets laptop memory physically and electrically plug into a
desktop DIMM slot — are on order from Amazon, total cost **~$25**, arriving the evening
of 2026-08-18.

**The question:** can those 2 free SO-DIMMs be added into the 2 empty slots — alongside
the existing 2 native UDIMM sticks, not replacing them — to reach 32GB total across 4
DIMMs, for the price of two $12ish adapter cards instead of buying a real 32GB desktop
kit outright?

**The bigger motivation behind trying at all:** this box is already the subject of an
earlier rigorous cost simulation about the 2026 DRAM shortage — see
[`ram-crisis-fleet-vs-single-machine-tco`](../software/ram-crisis-fleet-vs-single-machine-tco.md).
SO-DIMM pricing runs generally cheaper than equivalent desktop UDIMM even outside any
adapter-hack context, and that gap is exactly what the adapter trick is trying to
exploit during a period where desktop DDR5 pricing specifically is elevated. This entry
is the practical, single-box, does-it-actually-boot follow-up to that earlier
spreadsheet-level question — same underlying RAM-crisis motivation, this time tested
with real hardware instead of modeled.

**Why this is the highest-risk version of the idea, going in, not the safest:**

- SO-DIMM-to-DIMM adapters are, per the outlets that have actually tested them,
  "fundamentally a hack" — they introduce real signal degradation from the extra
  connector/trace path, and downclocking below the memory's rated speed is often needed
  to compensate. Hardware Canucks tested the concept across several current AM5/Intel
  platforms and found DDR5-4800 the most consistent target speed across the board, with
  anything higher depending heavily on the specific CPU memory controller and adapter
  revision (SOURCED: [KitGuru, "SODIMM to DIMM adapters could help gamers work around
  DDR5 memory shortage,"
  citing Hardware Canucks](https://www.kitguru.net/components/memory/joao-silva/sodimm-to-dimm-adapters-could-help-gamers-work-around-ddr5-memory-shortage/) —
  fetched 2026-08-17. That piece also cites a real cost angle: roughly 30% savings
  buying SO-DIMM + adapter versus equivalent desktop DIMM at then-current prices, and
  closes with "we wouldn't necessarily recommend running your PC this way").
- Populating all **4** DIMM slots on AM5 (as opposed to just 2) is a known separate
  stability cliff, independent of the adapter question — AM5's memory controller
  generally struggles to hold high speeds/tight timings once all 4 slots are populated
  at all, adapters or not. The specific figure driving this session's risk assessment —
  that fully-adapter-populated 4-DIMM AM5 configs were found "almost impossible to even
  boot at 4000 with reasonable latencies for that frequency" — came from research done
  earlier the same day this entry was started (AM5-specific enthusiast-forum testing
  threads); it wasn't re-confirmed against a live source when this entry was written, so
  treat it as ESTIMATED/recalled rather than a fresh citation. Worth re-verifying for
  real once the actual test result is in hand.
- **This specific configuration — 2 native UDIMM + 2 SO-DIMM-via-adapter, mixed in the
  same system, at mismatched rated speeds — was not covered by any sourced testing found
  at all**, adapter-specific or otherwise. Every reference above tested either matched
  adapter-only pairs, or matched native-DIMM 4-stick configs, not a mix of both
  simultaneously. Going in blind on the mixed case specifically is the actual gamble.

**Realistic expectation stated going in:** good chance the system fails to train memory
or POST at all with all 4 slots populated this way. If it does work — even at a
downclocked, loosened-timing fallback speed — that's a genuinely good result against the
odds, not the expected case.

**Why try it anyway:** the SO-DIMMs were already free, and the entire cost of finding
out is the ~$25 in adapters — cheap enough that a failed experiment isn't a real loss,
just an answer.

**Fallback plan if the 4-DIMM mix fails to boot:** the end state either way is reverting
to the current, already-stable 2x8GB native UDIMM configuration — this experiment isn't
meant to end in a new daily-driver config unless the full 32GB/4-DIMM mix actually
works. Before reverting, plan is to run a short side-experiment regardless: pull the 2
native UDIMM sticks entirely and boot the 2 SO-DIMM+adapter modules alone — a matched
2-DIMM adapter-only pair, the actual scenario the sourced testing above validated as
workable — purely to confirm the adapters themselves function at all. Worth doing even
knowing there's no lasting personal benefit either way (that pair isn't going to become
the daily config regardless of outcome) — it's still a real answer to a real question,
cheap to get since the hardware's already in hand at that point. That adapter-only pair
is not a candidate for ongoing daily use regardless of whether it posts; it's just a
data point on the way back to the native 2x8GB config, which is where the system lands
either way (unless the 4-DIMM mix succeeds, or a proper matched
2x16GB SO-DIMM kit gets bought separately later as a deliberate capacity upgrade).

## Execution

**Prep note, written before the test:** if the 4-DIMM mix fails to train memory
entirely, there's genuinely nothing to instrument from the software side — nothing
boots, so there's no OS and no log to capture. The one thing worth watching for anyway,
since it costs nothing: the board (ASUS B850 MAX GAMING WIFI) has a built-in **Q-LED
Core** — four single-color status LEDs (CPU / VGA / Boot / DRAM) that light up during
POST to show which stage it's stuck at (SOURCED: [ASUS product
page](https://www.asus.com/us/motherboards-components/motherboards/others/b850-max-gaming-wifi-w/),
fetched 2026-08-17 — no numeric Q-Code readout on this board, just the four zone LEDs).
If it fails, worth noting in this log: which LED is lit/stuck (a DRAM LED specifically
would confirm the failure is at memory training, as expected, versus something
unrelated going wrong first), and whether the board auto-retries at looser
timings/lower speed on repeated failed boots before giving up — many modern AM5 boards
do this automatically, and a fallback success at some much lower speed after a few
retries would itself be a real (if partial) result, not just a flat "didn't work."

**What actually happened, 2026-08-19:** installed both SO-DIMM-to-DIMM adapters (with
the SO-DIMMs seated in them) into the 2 empty slots, alongside the existing 2x8GB native
UDIMM in their original slots — the full 4-DIMM, 32GB mixed config described above.
Powered on. The board's internal ARGB lighting came up normally (that header is powered
independent of POST, so its own it isn't a useful signal either way), but nothing else
happened — no display output, no beep, no successful boot. The **Q-LED Core zone LEDs
were not legible in this case/board combo** — the only visible indicator was the power
button LED itself holding a steady, continuous flash the entire time, which does not
match the board's documented per-zone Q-LED behavior closely enough to say which stage
(CPU/VGA/Boot/DRAM) it was actually stuck at, or to distinguish a single failed attempt
from repeated auto-retries at looser timings. So: confirmed failure to POST, but the
stage-level diagnostic this section planned to lean on going in was not actually
obtainable from this rig — worth remembering for next time that the Q-LED Core isn't
reliably visible here without pulling the case open and looking directly at the board
zones near the DIMM slots, not just glancing at the front panel.

Reverted immediately: pulled both adapters/SO-DIMMs, restored the 2 native UDIMM to
their original slots, powered on — normal boot, confirmed via `dmidecode` post-boot
(both DIMM A/B populated at 8GB/4800 MT/s as before, adapter slots empty). The matched-
pair fallback test (2 SO-DIMM+adapter alone, no native UDIMM, the config the sourced
Hardware Canucks testing actually validated as workable) has not been run — reverting to
the known-good config was the priority for today, and that test is still a real open
question if this gets picked back up.

## Findings

- **The 4-DIMM mixed config (2 native UDIMM + 2 SO-DIMM-via-adapter) fails to POST on
  this board**, at least at stock settings (4800 MT/s, no manual downclocking or timing
  loosening attempted). This matches the "realistic expectation stated going in" above —
  a failure here was the likely outcome per the sourced research, not a surprise.
- No stage-level diagnosis was possible (see Execution) — can't confirm from this test
  alone whether it's specifically DRAM training failing (as the pre-test hypothesis
  assumed) versus something else. The steady power-LED flash with no Q-LED zone
  visibility just isn't enough signal to tell.
- No downclocking, timing-loosening, or single-adapter-pair (2-DIMM, no native UDIMM)
  test was attempted yet — those remain the actual open next steps, not ruled out by
  this result. This test only answers the highest-risk, most-ambitious version of the
  question (all 4 slots, mismatched pairs, stock speed); it doesn't say anything yet
  about whether the adapters work at all in a more modest, matched-pair config.
- System is back on the stable native 2x8GB UDIMM config as of this test, confirmed
  booting normally afterward.
