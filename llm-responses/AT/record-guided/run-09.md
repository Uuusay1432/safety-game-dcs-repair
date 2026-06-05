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

// -----------------------------------------------------------------------------
// Added controllable permission actions before requestLand / extendFlight.
// These actions synchronize with the corresponding descend actions suggested
// by the Additional Control Record.
// -----------------------------------------------------------------------------

Airplane(P=0) = (
    permitRequestLand[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    permitExtendFlight[P] -> extendFlight[P] -> End),

  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),

  LANDING[P] = (land[P] -> End),

  End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane]}  -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00	= [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01	= [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10	= [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11	= [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor   = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
	safety = 	{
				HeightMonitor_00,HeightMonitor_01,
				HeightMonitor_10,HeightMonitor_11,
				RampMonitor,
				RequestMonitor
				}

	controllable = {
				descend[Plane][Height],
				approach[Plane],
				permitRequestLand[Plane],
				permitExtendFlight[Plane]
				}

	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of the Changes

The Additional Control Record indicates that the uncontrollable actions:

* `requestLand.0`
* `extendFlight.0`

must be coordinated with controllable action `descend.1.0`, and similarly:

* `requestLand.1`
* `extendFlight.1`

must be coordinated with controllable action `descend.0.0`.

To realize this structurally while preserving the original safety properties and process structure, two new controllable permission actions were introduced:

* `permitRequestLand[P]`
* `permitExtendFlight[P]`

These actions are inserted immediately before the corresponding uncontrollable actions:

```fsp
permitRequestLand[P] -> requestLand[P]
permitExtendFlight[P] -> extendFlight[P]
```

The new actions were added to the controllable set so the synthesized controller can decide when these uncontrollable events are allowed to occur, effectively creating the additional synchronization/control points required by the safety game analysis.

No safety monitors or existing uncontrollable semantics were weakened or removed.
