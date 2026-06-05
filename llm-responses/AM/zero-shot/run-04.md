# Iteration 1

The unrealizability comes from a classic **environment-forces-illegal-choice** problem.

### Likely causes of unrealizability

1. **User 2 can uncontrollably choose any data** via `select_data[2][data]`, including private data (`2..5`), but safety requires:

   * `P_SECURITY_OK_02 = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})`
   * meaning user 2 may transfer **only public data (data 1)**.

   If the environment selects private data, the controller cannot later permit `dataTransfer`, yet it also cannot force a legal alternative.

2. **`ok/ng` happens before authorization is enforced.**
   Since `select_data` is already controllable, the natural repair is to gate it with explicit permission/confirmation so only legal selections are enabled.

3. Mutual exclusion (`P_NOT_OTHER_REQUEST_*`) is already satisfiable because `request` is serialized by the environment structure; no change needed there.

---

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

/* added controllable permission action */
set ControllableActions = {
    permit_select[SystemID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST
    = (request[ID] -> PERMISSION),

/* controller must explicitly permit a selection */
PERMISSION
    = (permit_select[ID] -> DATA_SELECTION),

DATA_SELECTION
    = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (ok[ID][data] -> OK[data]
     | ng[ID][data] -> NG[data]),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (filter[ID] -> GET_DATA[data]
     | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

---

### Explanation of changes

* Added **`permit_select[SystemID]`** as a new controllable synchronization action before `select_data`.
* New state `PERMISSION` forces the controller to **authorize data selection before the environment can choose data**.
* This lets the controller block illegal selections for **user 2** (private data 2–5), restoring synthesizability without weakening any safety property.
* All original process names, actions, and safety monitors are preserved.

# Iteration 2

The proposed repair with `permit_select` is unlikely to recover synthesizability because it does **not actually constrain which `select_data` action may occur after permission is granted**. Once the system reaches `DATA_SELECTION`, the environment can still execute any `select_data[2][d]` (including private data 2..5), recreating the original uncontrollable violation.

A more robust repair is to introduce an explicit **controller-authorized data selection handshake**, where the controller chooses the specific data item that may be selected, and the environment must confirm that exact choice. This preserves all safety properties while giving the controller sufficient authority to prevent illegal requests by user 2.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

/* added authorization handshake */
set ControllableActions = {
    authorize_data[SystemID][DataID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST
    = (request[ID] -> AUTHORIZATION),

/* controller chooses which data may be requested */
AUTHORIZATION
    = (authorize_data[ID][data:DataID] -> DATA_SELECTION[data]),

/* environment can only confirm the authorized choice */
DATA_SELECTION[data:DataID]
    = (select_data[ID][data] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (ok[ID][data] -> OK[data]
     | ng[ID][data] -> NG[data]),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (filter[ID] -> GET_DATA[data]
     | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of the changes

* Added a new controllable action:

  * `authorize_data[SystemID][DataID]`
* Inserted an authorization state between `request` and `select_data`.
* The controller now chooses a specific data item via `authorize_data[id][data]`.
* The environment can only perform the matching `select_data[id][data]` afterward.
* This prevents user 2 from uncontrollably selecting private data (2..5), allowing the controller to authorize only public data (`1`) for user 2.
* All original safety requirements are preserved unchanged. The repair only adds a synchronization mechanism that restores controller authority over the critical environment choice.

# Iteration 3

The remaining source of unrealizability is not only data selection. Even after introducing `authorize_data`, the environment can still force the controller into violating one of the following:

* `P_SECURITY_Admin = [](!reject[1][DataID])`
* `P_SECURITY_NG_02 = [](!reject[2][Public_Data])`

because after a selection the environment may nondeterministically reach a state where the only enabled transition is an uncontrollable decision that would eventually require a forbidden rejection. A standard DCS repair is to make the acceptance/rejection outcome explicitly controller-authorized before the environment commits to it.

The revised model therefore introduces a second synchronization step that lets the controller authorize the judgment outcome for the selected data item. This preserves all requirements while ensuring the environment cannot force an illegal reject/transfer combination.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {
    authorize_data[SystemID][DataID],
    authorize_ok[SystemID][DataID],
    authorize_ng[SystemID][DataID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST
    = (request[ID] -> AUTHORIZATION),

/* controller chooses which data may be requested */
AUTHORIZATION
    = (authorize_data[ID][data:DataID] -> DATA_SELECTION[data]),

/* environment can only select the authorized data */
DATA_SELECTION[data:DataID]
    = (select_data[ID][data] -> JUDGE_PERMISSION[data]),

/* controller authorizes the outcome */
JUDGE_PERMISSION[data:DataID]
    = (
          authorize_ok[ID][data] -> ok[ID][data] -> OK[data]
        | authorize_ng[ID][data] -> ng[ID][data] -> NG[data]
      ),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (
          filter[ID]   -> GET_DATA[data]
        | unfilter[ID] -> GET_DATA[data]
      ),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of the changes

1. **Data authorization handshake**

   * Added `authorize_data[id][data]`.
   * The controller chooses the specific data item that may be requested.
   * The environment can only execute the matching `select_data`.

2. **Judgment authorization handshake**

   * Added `authorize_ok[id][data]` and `authorize_ng[id][data]`.
   * The controller decides whether the selected request may proceed through the `ok` or `ng` branch before the environment commits to it.

3. **Safety preservation**

   * No safety monitor was changed.
   * User 2 can be restricted to public data only (`data=1`) before transfer becomes reachable.
   * Public data for user 2 can be routed through the authorized `ok` branch, avoiding the forbidden `reject[2][1]`.
   * User 1 can always avoid the forbidden `reject[1][data]` branch.

This repair keeps the original workflow and action structure while adding only the synchronization needed to prevent uncontrollable choices from forcing safety violations.

# Iteration 4

The previous repair over-constrains the model by making both `select_data` and the judgment outcome (`ok/ng`) controllable through duplicated authorization actions. In DCS repairs, a better approach is usually to identify the **actual uncontrollable source of the safety violation** and insert the minimal synchronization necessary.

Here, the key unrealizability source is that user 2 can reach forbidden transfers of non-public data. The controller already controls `ok`, `ng`, `reject`, `dataTransfer`, `filter`, and `process`. Therefore, the only genuinely dangerous uncontrollable choice is the environment's data selection. Once the selected data is known, the controller can choose the safe branch.

A minimal repair is therefore to keep the authorization handshake only around data selection and leave the original judgment structure intact.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

/* Added only the synchronization needed to control data choice */
set ControllableActions = {
    authorize_data[SystemID][DataID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST
    = (request[ID] -> AUTHORIZATION),

/* controller chooses which data may be selected */
AUTHORIZATION
    = (authorize_data[ID][data:DataID] -> DATA_SELECTION[data]),

/* environment confirms the authorized choice */
DATA_SELECTION[data:DataID]
    = (select_data[ID][data] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (
          ok[ID][data] -> OK[data]
        | ng[ID][data] -> NG[data]
      ),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (
          filter[ID]   -> GET_DATA[data]
        | unfilter[ID] -> GET_DATA[data]
      ),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of the changes

* Added a single controllable synchronization action:

  * `authorize_data[SystemID][DataID]`
* Inserted an authorization state between `request` and `select_data`.
* The controller first authorizes a specific data item.
* The environment can then only confirm that authorized item via `select_data[id][data]`.
* No safety property was modified.
* No additional judgment-authorizing actions were introduced because `ok`, `ng`, `reject`, and `dataTransfer` are already controllable and do not require further synchronization.

This preserves the original structure while removing the uncontrollable data-selection choice that can force safety violations.

# Iteration 5

A closer inspection shows that the current repair still contains a structural issue: making `authorize_data` controllable does **not** actually constrain the environment unless the original uncontrollable choice is removed from the environment's freedom.

In FSP/DCS, if `select_data` remains an independent action in the environment alphabet, introducing a preceding authorization state helps only if the environment is forced to follow the authorized value. The current parameterized state does that correctly, but there is still another source of unrealizability:

* User 1 may never execute `reject[1][d]` (`P_SECURITY_Admin`).
* User 2 may never execute `reject[2][1]` (`P_SECURITY_NG_02`).
* The environment can still force the system into the `NG` branch if the controller allows an unsafe data/request combination to reach a state where rejection is the only continuation.

A standard repair is to make the **request admission** itself controllable, allowing the controller to decide when a request may proceed into the workflow. This is a minimal permission mechanism that preserves the original structure and prevents the environment from forcing unsafe request sequences.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {
    permit_request[SystemID],
    authorize_data[SystemID][DataID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST
    = (request[ID] -> REQUEST_GRANTED),

REQUEST_GRANTED
    = (permit_request[ID] -> AUTHORIZATION),

/* controller chooses which data may be selected */
AUTHORIZATION
    = (authorize_data[ID][data:DataID] -> DATA_SELECTION[data]),

/* environment confirms the authorized choice */
DATA_SELECTION[data:DataID]
    = (select_data[ID][data] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (
          ok[ID][data] -> OK[data]
        | ng[ID][data] -> NG[data]
      ),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (
          filter[ID]   -> GET_DATA[data]
        | unfilter[ID] -> GET_DATA[data]
      ),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

* Added a new controllable permission action:

  * `permit_request[SystemID]`
* Inserted a request-admission synchronization step:

  * `request -> permit_request -> authorize_data -> select_data`
* Retained the previous data-authorization mechanism.
* Preserved all original safety properties unchanged.
* The controller now has authority both to admit a request and to authorize the selected data, preventing the environment from forcing unsafe request/data combinations that would eventually require forbidden rejection or transfer actions.

# Iteration 6

The main unrealizability source is the interaction between:

* `P_SECURITY_Admin = [](!reject[1][DataID])`
* `P_SECURITY_NG_02 = [](!reject[2][Public_Data])`
* the environment's ability to initiate requests and drive the protocol toward states where rejection becomes the only completion path.

The previous repairs added authorization around requests and data selection, but they still allow the environment to issue `request[1]` and `request[2]` independently. The two mutual-exclusion properties (`P_NOT_OTHER_REQUEST_01` and `P_NOT_OTHER_REQUEST_02`) are easier to satisfy if request admission itself is synchronized with the controller before the request is emitted.

A minimal and structurally clean repair is therefore to convert request generation into a request/permission handshake and retain the data-authorization mechanism.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}

/* new controllable synchronization actions */
set ControllableActions = {
    grant_request[SystemID],
    authorize_data[SystemID][DataID],
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

/* request must first be granted */
REQUEST
    = (grant_request[ID] -> request[ID] -> AUTHORIZATION),

/* controller chooses which data may be selected */
AUTHORIZATION
    = (authorize_data[ID][data:DataID] -> DATA_SELECTION[data]),

/* environment confirms the authorized choice */
DATA_SELECTION[data:DataID]
    = (select_data[ID][data] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (
          ok[ID][data] -> OK[data]
        | ng[ID][data] -> NG[data]
      ),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (
          filter[ID]   -> GET_DATA[data]
        | unfilter[ID] -> GET_DATA[data]
      ),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin
    = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02
    = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02
    = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01
    = [](request[1] ->
        X(!request[2] W
          (process[1]
          || reject[1][1]
          || reject[1][2]
          || reject[1][3]
          || reject[1][4]
          || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02
    = [](request[2] ->
        X(!request[1] W
          (process[2]
          || reject[2][1]
          || reject[2][2]
          || reject[2][3]
          || reject[2][4]
          || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

1. Added `grant_request[SystemID]` as a controllable permission action.
2. Replaced direct request generation with:

   ```
   grant_request[id] -> request[id]
   ```

   so the controller decides when a request may begin.
3. Retained the existing `authorize_data[id][data]` synchronization so the controller controls which data item may be selected.
4. Preserved all safety properties unchanged and kept the original workflow structure (`request → select_data → judgement → transfer/reject → process`).
