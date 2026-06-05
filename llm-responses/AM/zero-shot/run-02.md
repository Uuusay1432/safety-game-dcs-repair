# Iteration 1

The unrealizability comes from an **environment-driven premature commitment** problem.

### Likely cause of unrealizability

For `System(2)` (non-admin/general user), after `select_data[2][d]`, the environment can nondeterministically force:

* `ok[2][d]`, which may lead to `dataTransfer[2][d]`
* `ng[2][d]`, which may lead to `reject[2][d]`

But the safety requirements impose:

* `P_SECURITY_OK_02`: user 2 must **never transfer non-public data** (`d != 1`)
* `P_SECURITY_NG_02`: user 2 must **never reject public data** (`d = 1`)

So for:

* `d = 1`, environment could still force `ng[2][1]`, violating `P_SECURITY_NG_02`.
* `d = 2..5`, environment could force `ok[2][d]`, eventually requiring forbidden transfer.

Since `ok/ng` are **uncontrollable**, the controller cannot prevent this.
The standard repair is to insert a **controllable authorization/confirmation step before judgement**, allowing the controller to enable only the safe branch.

---

# Revised DCS Model

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

/* added controllable permission actions */
set ControllableActions = {
    select_data[SystemID][DataID],
    permit_ok[SystemID][DataID],
    permit_ng[SystemID][DataID],
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

REQUEST                =
        (request[ID] -> DATA_SELECTION),

DATA_SELECTION         =
        (select_data[ID][data:DataID] -> PERMISSION[data]),

/* inserted controller-mediated permission step */
PERMISSION[data:DataID] =
        (permit_ok[ID][data] -> OK_PATH[data]
        |permit_ng[ID][data] -> NG_PATH[data]),

OK_PATH[data:DataID]   =
        (ok[ID][data] -> OK[data]),

NG_PATH[data:DataID]   =
        (ng[ID][data] -> NG[data]),

NG[data:DataID]        =
        (reject[ID][data] -> REQUEST),

OK[data:DataID]        =
        FILTERING[data],

FILTERING[data:DataID] =
        (filter[ID] -> GET_DATA[data]
        |unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]  =
        (dataTransfer[ID][data] -> USE_DATA),

USE_DATA               =
        (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin   = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2]
          W (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1]
          W (process[2]
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

1. Added new controllable actions:

   * `permit_ok[ID][DataID]`
   * `permit_ng[ID][DataID]`

2. Inserted new state:

   * `PERMISSION[data]`

   after `select_data`, before environment can emit `ok/ng`.

3. `ok/ng` remain in the model (structure preserved), but now can occur **only after controller authorization**.

This restores synthesizability because the controller can enforce:

* for public data (`1`): only `permit_ok`
* for private data (`2..5`): only `permit_ng`

thus satisfying all original safety properties without weakening them.

# Iteration 2

The main source of unrealizability is still the uncontrollable generation of `ok`/`ng` outcomes after data selection. Although a permission state was introduced, the model currently makes **both `permit_ok` and `permit_ng` controllable**, while the subsequent uncontrollable `ok` or `ng` events remain enabled. This can still create mismatches between the controller's decision and the environment's subsequent behavior.

A more robust repair is to make the controller authorize the actual outcome before the environment proceeds. This preserves the original workflow while ensuring that the controller can prevent the unsafe combinations that violate:

* `P_SECURITY_OK_02` (user 2 transferring non-public data),
* `P_SECURITY_NG_02` (user 2 rejecting public data),
* and the mutual-request exclusion properties.

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

/* Added controllable authorization actions */
set ControllableActions = {
    select_data[SystemID][DataID],
    allow_ok[SystemID][DataID],
    allow_ng[SystemID][DataID],
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

REQUEST =
        (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
        (select_data[ID][data:DataID] -> JUDGEMENT[data]),

/* Controller authorizes which judgement branch may occur */
JUDGEMENT[data:DataID] =
        (allow_ok[ID][data] -> ok[ID][data] -> OK[data]
        |allow_ng[ID][data] -> ng[ID][data] -> NG[data]),

NG[data:DataID] =
        (reject[ID][data] -> REQUEST),

OK[data:DataID] =
        FILTERING[data],

FILTERING[data:DataID] =
        (filter[ID] -> GET_DATA[data]
        |unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID] =
        (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
        (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
        \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2]
          W (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1]
          W (process[2]
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

### Explanation of the repair

* Added only two new controllable actions:

  * `allow_ok[ID][DataID]`
  * `allow_ng[ID][DataID]`

* Kept the original process structure (`REQUEST → DATA_SELECTION → JUDGEMENT → OK/NG → ...`) intact.

* Inserted a controller-mediated authorization step before entering the `ok` or `ng` branch.

This allows the synthesized controller to:

* authorize only `allow_ok[2][1]` for public data,
* authorize only `allow_ng[2][2..5]` for non-public data,

thereby preventing both forbidden transfers and forbidden rejections while preserving all original safety requirements unchanged.


# Iteration 3

The remaining unrealizability is likely not only the `ok/ng` choice, but also the **inter-request exclusion requirements**:

* `P_NOT_OTHER_REQUEST_01`
* `P_NOT_OTHER_REQUEST_02`

These properties require that once one user issues a `request`, the other user must not issue a `request` until the first user's transaction finishes (`process` or `reject`).

In the current environment, `request[1]` and `request[2]` are uncontrollable and independent. Therefore, after `request[1]`, the environment can immediately produce `request[2]`, violating the specification before the controller has any opportunity to intervene.

A standard DCS repair is therefore:

1. Keep the authorization mechanism before the `ok/ng` decision.
2. Add a controller-mediated permission before each request so that concurrent requests can be serialized.
3. Preserve all safety properties unchanged.

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

    select_data[SystemID][DataID],

    allow_ok[SystemID][DataID],
    allow_ng[SystemID][DataID],

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

REQUEST =
    (permit_request[ID] ->
        request[ID] ->
        DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] ->
        JUDGEMENT[data]),

/* Controller authorizes the judgement outcome */
JUDGEMENT[data:DataID] =
    (allow_ok[ID][data] ->
        ok[ID][data] ->
        OK[data]

    |allow_ng[ID][data] ->
        ng[ID][data] ->
        NG[data]),

NG[data:DataID] =
    (reject[ID][data] ->
        REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (filter[ID] -> GET_DATA[data]
    |unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID] =
    (dataTransfer[ID][data] ->
        USE_DATA),

USE_DATA =
    (process[ID] ->
        REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/

ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
       \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2]
          W (process[1]
             || reject[1][1]
             || reject[1][2]
             || reject[1][3]
             || reject[1][4]
             || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1]
          W (process[2]
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

1. **Added `permit_request[ID]`** before each `request[ID]`.

   * This allows the controller to serialize requests and satisfy `P_NOT_OTHER_REQUEST_01` and `P_NOT_OTHER_REQUEST_02`.

2. **Retained the authorization stage before `ok/ng`**:

   * `allow_ok[ID][data]`
   * `allow_ng[ID][data]`

   This lets the controller prevent:

   * `reject[2][1]` (forbidden public-data rejection),
   * `dataTransfer[2][2..5]` (forbidden non-public transfers).

3. **Preserved all original process names, action names, and workflow structure** (`REQUEST → DATA_SELECTION → JUDGEMENT → OK/NG → ...`), introducing only minimal permission actions where uncontrollable environment choices were causing unrealizability.