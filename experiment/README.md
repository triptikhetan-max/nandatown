# Does averaged reputation survive when half the traders cheat?

Tripti Khetan (Harvard HGSE, LDIT) — MIT AI Studio Fall 2026, Step 2

## The question

The built-in `reputation` scenario runs 16 honest traders, 4 malicious
traders, and 1 observer. The observer scores agents from trade reports
(+1 good, -2 bad) and broadcasts an "untrusted" warning when an agent's
score hits -3. The trust layer is `score_average`.

I changed one setting in `task.config`: `malicious_fraction: 0.2 -> 0.5`.

**Hypothesis (written before running):** when half the traders are malicious,
averaged reputation stops separating honest from malicious. I expected
detection of cheaters to collapse, and possibly honest agents to be
falsely flagged.

## Surprise 1: the setting is silently overridden

Changing only `malicious_fraction` produced a **byte-identical trace**
(`rep_fraction_only.yaml`, same seed). Reading
`nest_core/scenarios_builtin/reputation.py` showed why: the factory
computes counts from `malicious_fraction`, then the scenario's `roles:`
block overwrites them. The built-in YAML pins honest=16 / malicious=4,
so the documented task.config knob is dead in the default scenario.

Fix that keeps the change to one governing setting: `reputation_majority.yaml`
removes the redundant `roles:` override so `malicious_fraction` actually
governs, keeping agents=21, seed=42, and everything else identical.

## Result

|                       | baseline (4/20 malicious) | majority (10/20 malicious) |
|-----------------------|---------------------------|----------------------------|
| cheaters flagged      | 3 of 4 (75%)              | 3 of 10 (30%)              |
| honest falsely flagged| 0                         | 0                          |
| messages sent         | 280                       | 250                        |
| about malicious       | good 8 / bad 10           | good 10 / bad 14           |

Detection collapsed from 75% to 30%: seven of ten cheaters were never
flagged. Several still carried negative scores, but none crossed the -3
warning threshold.

## Surprise 2: the mechanism was not the one I predicted

I expected honest agents to get smeared. False positives stayed at zero,
because these malicious agents cheat on trades but never file false
reports (confirmed in `MaliciousAgent`: it under-delivers, it does not
slander). The real mechanism is **evidence dilution**: with half the
honest witnesses gone, each cheater accumulates bad reports too slowly
to cross the -3 warning threshold inside 5 rounds. The system fails
open: absence of evidence reads as good standing.

## Testing the mechanism

If dilution is the mechanism, it makes a prediction: more rounds should
restore detection, because the observer just needs more evidence per
cheater. It does, cleanly:

| malicious_fraction 0.5, rounds -> | 5   | 10  | 15  | 25  |
|-----------------------------------|-----|-----|-----|-----|
| cheaters caught                   | 30% | 70% | 80% | 90% |
| honest falsely flagged            | 0   | 0   | 0   | 0   |

And a dose-response sweep at 5 rounds shows the collapse is smooth, not
a cliff: 75% caught at fraction 0.2, 67% at 0.3, 50% at 0.4, 30% at
0.5, 7% at 0.7. False positives are zero in every run.

## Takeaway

Raising the cheater share to half the traders does not corrupt averaged
reputation here; it
slows it. The scoring never falsely convicts, it just needs more
evidence per cheater than a short game provides. The threshold detector
fails open on time, not on truth. For real reputation systems the
lesson is that the honest-reporter supply sets the detection latency,
and any system evaluated over a short window can be defeated by
population thinning alone, with zero ratings fraud.

## Reproduce

```bash
uv sync
uv run nest run scenarios/reputation.yaml            # baseline
uv run nest run experiment/rep_fraction_only.yaml    # surprise 1: identical trace
uv run nest run experiment/reputation_majority.yaml  # the experiment
uv run nest inspect traces/rep_majority.jsonl
```

## Tools and help used (disclosure)

Built working alongside Claude Code (Claude Fable 5) throughout:
environment setup, reading the scenario source to chase Surprise 1,
trace analysis and sweep scripts, and drafting this README. The choice of
experiment, the hypothesis, and the final read of the results are mine;
I ran the scenarios and inspected the traces myself. No other human help.
