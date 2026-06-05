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

/*
 * Added controllable permission actions:
 *   allowExtend[P]
 *   allowRequest[P]
 *
 * These are inserted only before the uncontrollable actions
 * extendFlight[P] and requestLand[P], as suggested by the
 * Additional Control Record. They create new control points
 * that can synchronize with descend actions.
 */
Airplane(P=0) = (
    allowRequest[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h]
  | allowExtend[P]  -> extendFlight[P] -> End
  ),
  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),
  LANDING[P] = (land[P] -> End),
  End = (control.all -> Airplane).

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

ltl_property RequestMonitor = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

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
		allowRequest[Plane],
		allowExtend[Plane]
	}
	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).
controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of changes

* Introduced **two new controllable actions**: `allowRequest[P]` and `allowExtend[P]`.
* Inserted them immediately before the previously uncontrollable actions:

  * `requestLand[P]`
  * `extendFlight[P]`
* This realizes the Additional Control Record by giving the controller a **permission/synchronization point** before those uncontrollable choices, allowing coordination with existing `descend[...]` decisions.
* No safety property or existing uncontrollable action semantics were changed; only new control points were added.
