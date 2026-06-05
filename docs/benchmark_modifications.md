# Benchmark Modifications

This document summarizes the modifications applied to the benchmark models used
in the paper. Each modified model is designed to be unrealizable unless an
additional synchronization, suppression, or operational control point is added
to the environment model.

## AT: Air Traffic

The Air Traffic model represents landing coordination between aircraft. The
modified model adds a sequential-processing constraint: after one aircraft
starts a landing process, another landing request should not be accepted until
the current landing process is completed.

Files:

- Unrealizable model: `../models/unrealizable/AT-2-2.lts`
- Reference model: `../models/reference/AT-2-2-origin.lts`

## AG: Art Gallery

The Art Gallery model represents occupancy management across rooms and
corridors. The modified model delegates an admission decision that was
previously controlled by the system to the environment, making it possible for
an entry permission to occur before the controller can enforce the safety
condition.

Files:

- Unrealizable model: `../models/unrealizable/AG-2-2.lts`
- Reference model: `../models/reference/AG-2-4-realizable.lts`

Note: the reference file keeps the source artifact's original AG-2-4 naming.

## AM: Access Management

The Access Management model represents access control with resource constraints.
The modified model adds a mutual-exclusion rule: while one user's request is
being processed, the other user should not be able to start a new request.

Files:

- Unrealizable model: `../models/unrealizable/AM-2-5.lts`
- Reference model: `../models/reference/AM-2-5-origin.lts`

## BW: Bidding Workflow

The Bidding Workflow model represents a project evaluation workflow. The
modified model includes behavior for external reviewers and adds an exclusion
rule that prevents a new document from being assigned to another team while one
document is already under review.

Files:

- Unrealizable model: `../models/unrealizable/BW-2-2.lts`
- Reference model: `../models/reference/BW-2-2-origin.lts`

In the paper's evaluation, no Mixed State was detected for BW(2,2), so the
proposed method did not generate an additional-control record for this model.

## KIVA

The KIVA model represents cooperative warehouse robots. The modified model
delegates robot movement to the autonomous driving side and adds collision
avoidance constraints so that two robots should not occupy the same shelf or
work area at the same time.

Files:

- Unrealizable model: `../models/unrealizable/KIVA-N-2-robot.lts`
- Reference model: `../models/reference/KIVA-N-2-robot-origin.lts`

In the paper's evaluation, KIVA produced a very large number of additional
control candidates, making the current sequential intervention and resynthesis
strategy impractical for this model.
