# Iteration 1
The unrealizability comes from an **environment/controller race** around `requestLand`:

* `RequestMonitor` forbids a second `requestLand[p]` until `land[p]`.
* But in `Airplane`, after `land[P] -> End`, the environment can only restart via uncontrollable `control.all -> Airplane`.
* Since `control.all` is not synchronized with controller intent, one airplane may restart and issue a fresh `requestLand[P]` while another aircraft is still in a critical descent/approach sequence, making the height mutual-exclusion constraints (`HeightMonitor_*`) impossible to enforce.
* Also, the reset after landing is purely autonomous; there is no controller permission to re-enable a plane.

A standard repair is to add a **controllable release/acknowledgment action** before an airplane may restart. This lets the controller decide when a landed plane may re-enter the system, restoring synthesizability without weakening safety.

---

## Revised DCS Model

```fsp
//******************************************************************************
// Environment
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
  LANDING[P] = (land[P] -> WAIT_RESET[P]),
  WAIT_RESET[P] = (reset[P] -> End),
  End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane],
   land[Plane], reset[Plane]} -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

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
    [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
	safety = {
		HeightMonitor_00, HeightMonitor_01,
		HeightMonitor_10, HeightMonitor_11,
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

### Explanation of changes

1. Added `WAIT_RESET[P] = (reset[P] -> End)` after `land[P]`.
2. Added new **controllable** action `reset[Plane]`.
3. Included `reset[Plane]` and `land[Plane]` in `ResponseMonitor` so monitoring remains consistent.
4. Left all original safety properties unchanged.

This introduces a **controller-mediated restart permission**, preventing uncontrolled re-entry and recovering realizability.

# Iteration 2

A likely remaining cause of unrealizability is that `requestLand` is still **uncontrollable** and can be issued by multiple airplanes concurrently. The safety property

```fsp
[](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
```

requires that once any plane requests landing, no further `requestLand` event may occur until that same plane lands. Since the controller cannot disable `requestLand`, two airplanes can independently generate requests and immediately violate `RequestMonitor`.

A standard DCS repair is therefore to introduce a **controller-granted permission** before a plane may issue `requestLand`. This preserves the original landing workflow while allowing the controller to serialize requests.

## Revised DCS Model

```fsp
//******************************************************************************
// Environment
//******************************************************************************/

const N = 2
const K = 2

const Planes  = N
const Heights = K

range Plane  = 0..(Planes-1)
range Height = 0..(Heights-1)

Airplane(P=0) = (
    permitRequest[P] -> REQUESTING[P]
  |
    extendFlight[P] -> End
  ),

  REQUESTING[P] = (
    requestLand[P] -> descend[P][h:Height] -> Holding[h]
  ),

  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),

  LANDING[P] = (
    land[P] -> WAIT_RESET[P]
  ),

  WAIT_RESET[P] = (
    reset[P] -> End
  ),

  End = (
    control.all -> Airplane
  ).

ResponseMonitor = (
    requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor
  |
    { permitRequest[Plane],
      extendFlight[Plane],
      descend[Plane][Height],
      approach[Plane],
      land[Plane],
      reset[Plane]
    } -> ResponseMonitor
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

||Environment = (
    forall [p:Plane] Airplane(p)
    || ResponseMonitor
).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Changes made

* Added controllable action `permitRequest[P]`.
* Required each airplane to obtain permission before executing `requestLand[P]`.
* Kept all original safety properties unchanged.
* Preserved the original descent/approach/landing/reset structure.
* Retained the previously added `reset[P]` permission step to avoid uncontrolled re-entry after landing.

This repair gives the controller control over the initiation of landing requests, allowing it to satisfy `RequestMonitor` while still enforcing the height-separation and runway-approach safety requirements.
