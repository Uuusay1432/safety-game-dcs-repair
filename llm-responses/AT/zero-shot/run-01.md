# Iteration 1

The unrealizability comes from an **uncontrollable `requestLand[p]` race**:

* `requestLand[p]` is **environment-controlled**.
* `RequestMonitor = [](requestLand[p] -> X(!requestLand[p] W land[p]))` forbids a second request for the same plane before landing.
* But `Airplane.End = (control.all -> Airplane)` allows the environment to restart a plane independently after `extendFlight[p]`, so a fresh `requestLand[p]` can occur before the controller has any chance to force `land[p]` (since `land` itself is uncontrollable in the plant).
* This makes synthesis fail because the controller cannot prevent the environment from reissuing requests.

The standard repair is to add a **controller-mediated reset permission** before recycling an airplane. This preserves all safety monitors while restoring realizability.

## Revised DCS Model

```fsp
//******************************************************************************
// Environment (Repaired)
//******************************************************************************/

const N = 2
const K = 2

const Planes  = N
const Heights = K

range Plane  = 0..(Planes-1)
range Height = 0..(Heights-1)

Airplane(P=0) = (
    requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlight[P] -> End),

  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),

  LANDING[P] = (land[P] -> End),

  // repaired: controller must explicitly permit recycle
  End = (reset[P] -> control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane], descend[Plane][Height], approach[Plane]} -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] =
    <requestLand[p], descend[p][Height]>

ltl_property HeightMonitor_00 =
    [](descend[0][0] -> X(!descend[Plane][0] W land[0]))

ltl_property HeightMonitor_01 =
    [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))

ltl_property HeightMonitor_10 =
    [](descend[1][0] -> X(!descend[Plane][0] W land[1]))

ltl_property HeightMonitor_11 =
    [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor =
    [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))

ltl_property RampMonitor =
    [](approach[Plane] -> X(!approach[Plane] W land[Plane]))

//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
    safety = {
        HeightMonitor_00,
        HeightMonitor_01,
        HeightMonitor_10,
        HeightMonitor_11,
        RampMonitor,
        RequestMonitor
    }
    controllable = {
        descend[Plane][Height],
        approach[Plane],
        reset[Plane]
    }
    nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).
controller ||TraditionalController = (Environment)~{Goal}.
```

### Summary of changes

* Added new controllable action: `reset[p]`.
* Changed `End = (control.all -> Airplane)` to `End = (reset[p] -> control.all -> Airplane)`.

This is a minimal **permission/synchronization repair**: the controller now decides when a plane may re-enter service, preventing uncontrollable premature `requestLand` events and restoring synthesizability.

# Iteration 2

The previous repair removes one source of unrealizability (recycling after `End`), but there is still a likely uncontrollable violation of `RequestMonitor`.

`requestLand[p]` is an uncontrollable environment action. Even with `reset[p]`, after a plane chooses the `requestLand[p]` branch and performs the first controlled descent, the environment can eventually return to a fresh `Airplane(p)` instance and issue another `requestLand[p]` before the corresponding `land[p]` has occurred. Since the controller cannot disable `requestLand[p]`, a safer repair is to introduce an explicit controller-granted permission before every landing request is issued.

This converts `requestLand[p]` into a synchronized request protocol while preserving all original safety monitors unchanged.

## Revised DCS Model

```fsp
//******************************************************************************
// Environment (Repaired)
//******************************************************************************/

const N = 2
const K = 2

const Planes  = N
const Heights = K

range Plane  = 0..(Planes-1)
range Height = 0..(Heights-1)

Airplane(P=0) = (
    permitRequest[P] ->
        requestLand[P] ->
        descend[P][h:Height] ->
        Holding[h]
  |
    extendFlight[P] -> End
),

Holding[h:Height] =
    if (h>0) then (
        descend[P][h-1] -> Holding[h-1]
    )
    else (
        approach[P] -> LANDING[P]
    ),

LANDING[P] = (
    land[P] -> End
),

End = (
    reset[P] ->
    control.all ->
    Airplane
).

ResponseMonitor = (
    requestLand[p:Plane] ->
        descend[p][Height] ->
        ResponseMonitor
  |
    {extendFlight[Plane],
     descend[Plane][Height],
     approach[Plane]} ->
        ResponseMonitor
).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] =
    <requestLand[p], descend[p][Height]>

ltl_property HeightMonitor_00 =
    [](descend[0][0] -> X(!descend[Plane][0] W land[0]))

ltl_property HeightMonitor_01 =
    [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))

ltl_property HeightMonitor_10 =
    [](descend[1][0] -> X(!descend[Plane][0] W land[1]))

ltl_property HeightMonitor_11 =
    [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor =
    [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))

ltl_property RampMonitor =
    [](approach[Plane] -> X(!approach[Plane] W land[Plane]))

//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
    safety = {
        HeightMonitor_00,
        HeightMonitor_01,
        HeightMonitor_10,
        HeightMonitor_11,
        RampMonitor,
        RequestMonitor
    }
    controllable = {
        permitRequest[Plane],
        descend[Plane][Height],
        approach[Plane],
        reset[Plane]
    }
    nonblocking
}

||Environment =
    (forall [p:Plane] Airplane(p) || ResponseMonitor).

controller ||TraditionalController =
    (Environment)~{Goal}.
```

### Changes

1. Added a new controllable synchronization action:

   * `permitRequest[p]`

2. Modified the landing-request branch:

   ```fsp
   requestLand[P] -> ...
   ```

   becomes

   ```fsp
   permitRequest[P] -> requestLand[P] -> ...
   ```

3. Kept the existing recycle permission:

   ```fsp
   reset[P] -> control.all -> Airplane
   ```

This repair introduces explicit controller authorization before each uncontrollable landing request and before recycling an aircraft, while leaving all safety requirements unchanged.
