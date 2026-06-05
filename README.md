# safety-game-dcs-repair

Artifacts for the paper:

**Safety-Game-Based Additional Control Candidate Suggestion for Repairing Unrealizable Discrete Controller Synthesis Models**

This repository contains the benchmark models and reference models used to
evaluate a repair-support method for unrealizable Discrete Controller Synthesis
(DCS) models. The method analyzes the losing region of a safety game, detects
Mixed States where controllable and uncontrollable actions compete, and suggests
additional control points to insert before selected uncontrollable actions.

## Repository Layout

```text
models/
  unrealizable/   Unrealizable DCS benchmark models used in the evaluation
  reference/      Reference or original models for comparison
prompts/
  templates/      Prompt templates used for the LLM repair experiment
  filled/         Prompt files prepared for LLM input
llm-responses/    Raw LLM response logs for each model, condition, and run
docs/
  benchmark_modifications.md
  llm_experiment.md
```

## Benchmarks

The evaluation uses five benchmark families:

| Model | Unrealizable model | Reference model |
| --- | --- | --- |
| AT(2,2) | `models/unrealizable/AT-2-2.lts` | `models/reference/AT-2-2-origin.lts` |
| AG(2,2) | `models/unrealizable/AG-2-2.lts` | `models/reference/AG-2-4-realizable.lts` |
| AM(2,5) | `models/unrealizable/AM-2-5.lts` | `models/reference/AM-2-5-origin.lts` |
| BW(2,2) | `models/unrealizable/BW-2-2.lts` | `models/reference/BW-2-2-origin.lts` |
| KIVA | `models/unrealizable/KIVA-N-2-robot.lts` | `models/reference/KIVA-N-2-robot-origin.lts` |

Details of the modifications applied to each base model are described in
[`docs/benchmark_modifications.md`](docs/benchmark_modifications.md).

## LLM Repair Experiment

The LLM-based repair experiment compares two conditions:

| Paper condition | Repository directory | Meaning |
| --- | --- | --- |
| Zero-shot | `llm-responses/*/zero-shot/` | The LLM receives only the current DCS model |
| Record-guided | `llm-responses/*/record-guided/` | The LLM receives the DCS model and an additional-control record |

Prompt templates are stored under `prompts/templates/`, and the prepared prompt
files used for LLM input are stored under `prompts/filled/`. Response logs are
organized by model and condition, with ten runs per condition:

```text
llm-responses/
  AT/{zero-shot,record-guided}/run-01.md ... run-10.md
  AM/{zero-shot,record-guided}/run-01.md ... run-10.md
  AG/{zero-shot,record-guided}/run-01.md ... run-10.md
```

See [`docs/llm_experiment.md`](docs/llm_experiment.md) for details.

## Reproduction

The models are written in the FSP/LTS notation used by MTSA.

1. Install MTSA (Modal Transition System Analyzer) from:
   <https://mtsa.dc.uba.ar/>
2. Open one of the `.lts` files under `models/unrealizable/`.
3. Run controller synthesis for the controller declared at the bottom of the
   selected file.
4. Compare the synthesis result with the corresponding file under
   `models/reference/`.

The unrealizable models are expected to fail controller synthesis under the
specified safety requirements. The reference models are included to make the
intended realizable behavior or original benchmark structure easier to inspect.

## Expected Results

The paper reports the following high-level outcomes:

| Model | Result reported in the paper |
| --- | --- |
| AT(2,2) | Additional-control records recovered realizability |
| AM(2,5) | Additional-control records recovered realizability |
| AG(2,2) | Additional-control records recovered realizability |
| BW(2,2) | No Mixed State was detected, so no record was generated |
| KIVA | Candidate explosion made sequential repair impractical |

In the LLM-based repair experiment, using additional-control records improved
first-attempt success from 19/30 to 28/30 and reduced the average number of
iterations to success from 1.63 to 1.07.
