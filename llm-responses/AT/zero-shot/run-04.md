# Iteration 1

The unrealizability comes from an **environment-forced `requestLand` race**.

### Likely cause

`RequestMonitor = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))`

This means: once a plane issues `requestLand[p]`, **no new `requestLand[*]` may occur until that same plane lands**.

But in the original environment, each `Airplane(p)` can independently do:

```fsp
requestLand[p] -> ...
```

Since `requestLand` is **uncontrollable** (not listed in `controllable`), the controller cannot stop plane 1 from issuing `requestLand[1]` while plane 0 is still descending/approaching. That directly violates `RequestMonitor`, making synthesis impossible.

The repair is therefore to add an **environment-side permission/synchronization gate** so only one aircraft may issue `requestLand` at a time, and permission is released after `land[p]` (or `extendFlight[p]`, since that terminates the request cycle).

This preserves all original safety properties and minimally changes only the environment.

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
    requestPermit[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlight[P] -> releasePermit[P] -> End),
  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),
  LANDING[P] = (land[P] -> releasePermit[P] -> End),
  End = (control.all -> Airplane).

/* mutual-exclusion gate for landing requests */
RequestGate = Free,
  Free = (
      requestPermit[p:Plane] -> Busy[p]
  ),
  Busy[p:Plane] = (
      releasePermit[p] -> Free
  );

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane]} -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00 = [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01 = [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10 = [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11 = [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor   = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor      = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

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
		requestPermit[Plane]
	}
	nonblocking
}

||Environment =
   (forall [p:Plane] Airplane(p)
    || RequestGate
    || ResponseMonitor).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Summary of changes

1. Added **`requestPermit[p]`** before each `requestLand[p]`.
2. Added **`releasePermit[p]`** after `land[p]` and after `extendFlight[p]`.
3. Added **`RequestGate`** enforcing mutual exclusion on landing requests.
4. Made only the new permission action (`requestPermit`) controllable; all original structure and safety monitors remain unchanged.

This restores synthesizability by preventing uncontrollable overlapping landing requests.

# Iteration 2

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
    requestPermit[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlight[P] -> End),
  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),
  LANDING[P] = (
      land[P] -> landingComplete[P] -> End
  ),
  End = (control.all -> Airplane).

/* Gate that allows only one active landing request at a time.
   Permission is released only after the corresponding landing completes. */
RequestGate = Free,
  Free = (
      requestPermit[p:Plane] -> Busy[p]
  ),
  Busy[p:Plane] = (
      landingComplete[p] -> Free
  ).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane], landingComplete[Plane]}
      -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00 = [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01 = [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10 = [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11 = [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor   = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor      = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

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
		requestPermit[Plane]
	}
	nonblocking
}

||Environment =
   (forall [p:Plane] Airplane(p)
    || RequestGate
    || ResponseMonitor).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation

The main source of unrealizability is `RequestMonitor`, which forbids any new `requestLand[*]` event between a plane's `requestLand[p]` and its eventual `land[p]`. Since `requestLand` is uncontrollable, the environment must prevent overlapping requests.

The repair introduces a synchronization mechanism:

* `requestPermit[p]` (already present and controllable) must be granted before `requestLand[p]`.
* A `RequestGate` allows only one permit at a time.
* The gate is released only after `landingComplete[p]`, which occurs immediately after `land[p]`.

This ensures that no second aircraft can issue `requestLand` before the first aircraft has landed, satisfying `RequestMonitor` without weakening any safety requirement. The original airplane behavior, action names, and safety properties are preserved, with only a minimal completion handshake added to synchronize permit release with successful landing.
