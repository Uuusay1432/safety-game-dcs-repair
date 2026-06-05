# LLM Repair Experiment

This directory documents the artifacts used in the LLM-based environment-model
repair experiment.

## Conditions

The paper evaluates two prompting conditions:

- Zero-shot: the prompt includes only the current unrealizable DCS model.
- Record-guided: the prompt includes the current DCS model and an
  additional-control record generated from the safety-game analysis.

In the source working repository, these conditions were named `noguide` and
`prop` or `proposal`. In this public artifact repository, they are normalized to
`zero-shot` and `record-guided`.

## Prompt Files

Prompt templates:

- `../prompts/templates/zero-shot.txt`
- `../prompts/templates/record-guided.txt`

Prepared prompt files:

- `../prompts/filled/zero-shot.txt`
- `../prompts/filled/record-guided.txt`

The templates contain placeholders for the original DCS model and, for the
record-guided condition, the additional-control record.

## Response Logs

Raw LLM responses are organized by model, condition, and run:

```text
../llm-responses/
  AT/
    zero-shot/run-01.md ... run-10.md
    record-guided/run-01.md ... run-10.md
  AM/
    zero-shot/run-01.md ... run-10.md
    record-guided/run-01.md ... run-10.md
  AG/
    zero-shot/run-01.md ... run-10.md
    record-guided/run-01.md ... run-10.md
```

These logs preserve the model text and explanations returned by the LLM. They
are included to support inspection of the repair strategies used in the
zero-shot and record-guided conditions.
