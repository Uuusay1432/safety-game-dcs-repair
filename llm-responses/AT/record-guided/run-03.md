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
 *  - permitReqExt_fromDescend[P] : before requestLand[P]/extendFlight[P]
 *    when the airplane has just been reset after descend[P][0]
 *
 *  - permitReqExt_fromApproach[P] : before requestLand[P]/extendFlight[P]
 *    when the airplane has just been reset after approach[P]
 *
 * These are made controllable and inserted immediately before the
 * previously uncontrollable requestLand/extendFlight actions.
 */

Airplane(P=0) = Start[P],

Start[P] = (
    permitReqExt_fromDescend[P] -> Decision[P] |
    permitReqExt_fromApproach[P] -> Decision[P]
),

Decision[P] = (
    requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlight[P] -> End
),

Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),

LANDING[P] = (land[P] -> End),

End = (control.all -> Start[P]).


ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane]} -> ResponseMonitor
).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00 = [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01 = [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10 = [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11 = [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor    = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

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
		permitReqExt_fromDescend[Plane],
		permitReqExt_fromApproach[Plane]
	}
	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).
controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of changes

* Added **two new controllable actions**:

  * `permitReqExt_fromDescend[P]`
  * `permitReqExt_fromApproach[P]`

* These are inserted **immediately before** the previously uncontrollable actions `requestLand[P]` and `extendFlight[P]`, turning them into explicit controller-mediated permission points.

* The split directly realizes the Additional Control Record:

  * after **`descend[P][0]`**, controller can gate `requestLand[P]` / `extendFlight[P]` (matches `descend.1.0` and `descend.0.0`);
  * after **`approach[P]`**, controller can gate `requestLand[P]` / `extendFlight[P]` (matches `approach.1`).

* All original safety properties and original uncontrollable action meanings are preserved.
