# ListSelector & Carrier Flow — Application Startup to Flights Navigation

Now that all four files have been shown, the flow becomes much clearer.

The important thing is: `ListSelector` is created first, but it does not have the Carrier List yet. It prepares two waiting mechanisms. Later, the Carrier controller gives it the List.

---

## Complete Flow (High Level)

```
Application starts
        ↓
Component.js → init()
        ↓
new ListSelector()
        ↓
ListSelector constructor runs
        ↓
Creates Promise #1
"_oWhenListHasBeenSet"
        ↓
Creates Promise #2
"oWhenListLoadingIsDone"
        ↓
Component continues
        ↓
Router initialized
        ↓
Overview/Carrier view is displayed
        ↓
CarrierController.onInit()
        ↓
Gets Carrier List
        ↓
onBeforeFirstShow()
        ↓
setBoundMasterList(oList)
        ↓
ListSelector now knows the Carrier List
        ↓
Wait for OData dataReceived
        ↓
Carrier data arrives
        ↓
Get first Carrier
        ↓
oWhenListLoadingIsDone → RESOLVED
        ↓
CarrierController._onListMatched()
        ↓
Gets first Carrier
        ↓
Navigate to Flights
```

---

## Now Let's Connect the Code

### 1. Component.js starts everything

This is the first important line:

```javascript
this.oListSelector = new ListSelector();
```

So UI5 creates a new `ListSelector`.

```
Component
   │
   └── oListSelector
           ↓
      ListSelector object
```

At this moment, the `ListSelector` constructor runs.

---

### 2. ListSelector constructor runs

First:

```javascript
this._oWhenListHasBeenSet = new Promise(...)
```

This creates:

```
_oWhenListHasBeenSet
        ↓
    Promise ⏳
```

Its meaning is:

> "I am waiting for someone to give me the Master List."

Then:

```javascript
this.oWhenListLoadingIsDone = new Promise(...)
```

creates another:

```
oWhenListLoadingIsDone
        ↓
    Promise ⏳
```

Its meaning is:

> "I am waiting for the Master List's data to finish loading."

So after the constructor:

```
ListSelector
│
├── _oWhenListHasBeenSet
│       ↓
│    waiting for List
│
├── oWhenListLoadingIsDone
│       ↓
│    waiting for List data
│
└── _fnResolveListHasBeenSet
        ↓
    function used later
```

---

### 3. Why does Promise #2 depend on Promise #1?

Look at:

```javascript
this._oWhenListHasBeenSet
    .then(function (oList) {
```

This means:

> "When the Master List has been provided, THEN do this."

So:

```
Promise #1
"Do we have the List?"
       ↓
      YES
       ↓
Promise #2 starts watching
"Has the List data arrived?"
```

That's why there are two stages.

---

### 4. Then CarrierController starts

Now look at:

```javascript
onInit: function () {
    var oList = this.byId("list");
    this._oList = oList;
```

The Carrier controller gets the Carrier List.

```
CarrierController
       │
       └── _oList
             ↓
        Carrier List
```

But `ListSelector` doesn't know about this List yet.

---

### 5. onBeforeFirstShow gives the List to ListSelector

This code:

```javascript
this.getView().addEventDelegate({
    onBeforeFirstShow: function () {
        this.getOwnerComponent().oListSelector.setBoundMasterList(this._oList);
    }.bind(this)
});
```

Eventually runs before the Carrier view is shown.

It calls:

```javascript
setBoundMasterList(this._oList);
```

So now:

```
CarrierController
       │
       │ gives List
       ↓
ListSelector
```

---

### 6. setBoundMasterList() does two things

```javascript
setBoundMasterList: function(oList) {
    this._oList = oList;
    this._fnResolveListHasBeenSet(oList);
}
```

**First:**

```javascript
this._oList = oList;
```

means:

> "ListSelector, remember this Carrier List."

Now:

```
ListSelector
     │
     └── _oList → Carrier List
```

**Second:**

```javascript
this._fnResolveListHasBeenSet(oList);
```

means:

> "The List is available now. Stop waiting."

So Promise #1 becomes:

```
_oWhenListHasBeenSet
        ↓
     RESOLVED ✅
        ↓
   Carrier List
```

---

### 7. Now Promise #2 can continue

Remember this?

```javascript
this._oWhenListHasBeenSet
    .then(function(oList) {
```

Because Promise #1 has now been resolved, the code inside `.then()` can run.

It receives:

```
oList
  ↓
Carrier List
```

Then it does:

```javascript
oList.getBinding("items").attachEventOnce("dataReceived", ...)
```

In simple English:

> "Now that I have the List, tell me when its backend data arrives."

---

### 8. Backend data arrives

Suppose backend returns:

```
AA
AZ
LH
SQ
```

The `dataReceived` event fires.

Then:

```javascript
var oFirstListItem = oList.getItems()[0];
```

gets:

```
AA
```

Then:

```javascript
fnResolve({
    list: oList,
    oFirstListItem: oFirstListItem
})
```

resolves Promise #2.

So:

```
oWhenListLoadingIsDone
          ↓
      RESOLVED ✅
          ↓
{
   list: Carrier List,
   oFirstListItem: AA
}
```

---

### 9. CarrierController is waiting for this

Look at:

```javascript
_onListMatched: function () {
    this.getListSelector().oWhenListLoadingIsDone.then(
        function (mParams) {
```

This means:

> "When the Carrier List has finished loading, tell me the result."

When Promise #2 resolves, `mParams` contains:

```javascript
{
    list: oList,
    oFirstListItem: first item
}
```

So:

```javascript
mParams.oFirstListItem
```

is the first Carrier.

Then:

```javascript
var sObjectId =
    mParams.oFirstListItem
        .getBindingContext()
        .getProperty("Carrid");
```

gets its Carrier ID.

For example:

```
First Carrier
     ↓
AA
     ↓
Carrid = AA
```

Then:

```javascript
this._navigateToCarrierDetails(sObjectId, true);
```

navigates to:

```
Flights for AA
```

---

### 10. Then Flights controller takes over

The route:

```
flights
```

is matched.

So:

```javascript
_onObjectMatched: function(oEvent)
```

runs.

It gets:

```javascript
var oArgs = oEvent.getParameter("arguments");
```

For example:

```
carrid = AA
```

Then:

```javascript
oView.bindElement({
    path: "/UX_C_Carrier_TP('" + this._sCarrierId + "')"
});
```

binds the Flight detail view to:

```
/UX_C_Carrier_TP('AA')
```

---

## The Entire Application Flow

This is the picture to remember:

```
                    Component.js
                         │
                         ↓
                 new ListSelector()
                         │
                         ↓
              ListSelector constructor
                         │
                ┌────────┴────────┐
                ↓                 ↓
           Promise #1        Promise #2
           "Wait for List"   "Wait for data"
                │                 │
                │                 │
                ↓                 │
          CarrierController       │
                │                 │
                ↓                 │
          Gets Carrier List        │
                │                 │
                ↓                 │
       setBoundMasterList()        │
                │                 │
                ↓                 │
        List is now available      │
                │                 │
                └────────────────→ │
                                  ↓
                         Wait for dataReceived
                                  │
                                  ↓
                           Backend responds
                                  │
                                  ↓
                           First Carrier found
                                  │
                                  ↓
                         Promise #2 resolved
                                  │
                                  ↓
                    _onListMatched() continues
                                  │
                                  ↓
                       Get first Carrier ID
                                  │
                                  ↓
                        Navigate to flights
                                  │
                                  ↓
                     FlightsController starts
                                  │
                                  ↓
                        bindElement(Carrier)
```

---

## The Key Idea

There are two promises because there are two different things to wait for:

**1st waiting:**

> "Do I have the List?"
>
> YES

**2nd waiting:**

> "Has the List's backend data arrived?"
>
> YES

Then:

> "Give me the first Carrier."

`ListSelector` is basically a helper object that manages this waiting and selection logic.
