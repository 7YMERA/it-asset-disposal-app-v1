# Key formulas

A few pieces of the Power Fx that do the real work. Everything here is copied from [`src/`](../src).

## Recording every user at startup

[`App.pa.yaml`](../src/App.pa.yaml): `App.OnStart`

```powerfx
// 1. Set the logged-in user's details
Set(varUserEmail, User().Email);
Set(varUserName, User().FullName);

// 3. Check if this user already exists in the Requestor List
Set(
    varIsDuplicateRequestor,
    !IsBlank(LookUp('Requestor', Email = varUserEmail))
);

// 4. If not already listed, add them
If(
    !varIsDuplicateRequestor,
    Patch('Requestor', Defaults('Requestor'), { Name: varUserName, Email: varUserEmail })
);
```

This replaced a separate "request access" step: anyone who opens the app can create a case.

## Sending an approver to the right screen

[`Login.pa.yaml`](../src/Login.pa.yaml): the Approver button

```powerfx
// Check if user is 1st or 2nd approver by scanning the whole SharePoint list
ClearCollect(
    colUserApproverRoles,
    Filter(
        'ITAssetDisposal-Case',
        '1stApproverEmail' = varLoggedInEmail || '2ndApproverEmail' = varLoggedInEmail
    )
);
Set(varIsFirstApprover,  !IsEmpty(Filter(colUserApproverRoles, '1stApproverEmail' = varLoggedInEmail)));
Set(varIsSecondApprover, !IsEmpty(Filter(colUserApproverRoles, '2ndApproverEmail' = varLoggedInEmail)));

If(
    varIsFirstApprover,
    Navigate(Approver_ViewRequest1_1stLevel, ScreenTransition.Fade),
    If(varIsSecondApprover, Navigate(Approver_ViewRequest1_2ndLevel, ScreenTransition.Fade))
)
```

The filter result is loaded into a collection once, and the two checks run against that collection
instead of going back to SharePoint each time.

## Saving a new case with its items and watchlist

[`Requestor_NewRequest_1.pa.yaml`](../src/Requestor_NewRequest_1.pa.yaml): the Save button

```powerfx
// Step 1: Create new Case
Set(
    varNewCase,
    Patch(
        'ITAssetDisposal-Case',
        Defaults('ITAssetDisposal-Case'),
        {
            LocationID: DropdownLocation.Selected.Value,
            requestorName: varUserName,
            RequestorEmail: varUserEmail,
            Location: Label4_12.Text,
            '1stApproverName': Label3.Text,
            '1stApproverEmail': Label3_1.Text,
            '2ndApproverName': Label3_4.Text,
            '2ndApproverEmail': Label3_5.Text,
            Status: "Draft",
            Year: Year(Now()),
            UID: Text(Now(), "yyyymmddhhmmssfff")
        }
    )
);

// Step 2: Get ID of new case
Set(varCaseID, varNewCase.ID);

// Step 3: Submit colAssetItems to SharePoint and link to case
ForAll(
    colAssetItems,
    Patch(
        'ITAssetDisposal-Item',
        Defaults('ITAssetDisposal-Item'),
        {
            caseNo: varCaseID,
            assetType: ThisRecord.assetType,
            Items: ThisRecord.item,
            assetModel: ThisRecord.model,
            assetBrand: ThisRecord.manufacturer,
            SerialNum: ThisRecord.serial,
            Quantity: If(IsBlank(ThisRecord.quantity), "-", ThisRecord.quantity),
            AssetTag: ThisRecord.tag,
            UID: ThisRecord.uid
        }
    )
);

// Step 4: Submit colCurrentCaseWatchlist to SharePoint
ForAll(
    colCurrentCaseWatchlist,
    Patch(
        'ITAssetDisposal-Watchlists',
        Defaults('ITAssetDisposal-Watchlists'),
        { caseID: varCaseID, WatchlistEmail: ThisRecord.WatchlistEmail,
          WatchlistName: ThisRecord.WatchlistName, UID: ThisRecord.UID }
    )
);
```

Items are built up in a local collection while the requestor works, then written in one go when they
press Save. Leaving the screen before saving throws the list away, which is why the app asks for
confirmation first.

## Editing a draft without hitting delegation limits

[`Requestor_ViewRequest.pa.yaml`](../src/Requestor_ViewRequest.pa.yaml): saving edits to a case

```powerfx
// Remove items the user deleted
ForAll(colDeletedItems, RemoveIf('ITAssetDisposal-Item', UID = ThisRecord.UID));
Clear(colDeletedItems);

// Step 4: Refresh SharePoint items for the current case
ClearCollect(
    colSharePointItems,
    Filter('ITAssetDisposal-Item', caseNo = Text(varSelectedCase.ID))
);

// Step 5: Rebuild combined gallery data
ClearCollect(colCombinedAssets, colSharePointItems);
```

Power Apps can only push some filters down to SharePoint ("delegation"). Anything it can't delegate
only sees the first few hundred rows, so cases could silently go missing once the lists grew. The fix
was to load just the current case's items into local collections and do the rest of the filtering
and combining there.
