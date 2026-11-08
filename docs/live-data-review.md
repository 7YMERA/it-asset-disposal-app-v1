# What the live lists show

The source in [`src/`](../src) is an earlier copy of the app. Exports of the real SharePoint lists, taken at
the end of the internship, show how far the final version actually got and how it was used. This page
summarises those exports. Names, emails and list addresses are left out; the counts are exact.

## Size

| List | Rows | Notes |
|---|---|---|
| ITAssetDisposal-Case | 46 | IDs 1 to 61, created 19 June to 5 August 2025 |
| ITAssetDisposal-Item | 97 | 54 assets, 39 non-assets, 4 with no type |
| ITAssetDisposal-Location | 20 | 15 Malaysia, 3 Brunei, 2 test bases |

About a third of the cases (16) were raised against test bases, which is how every approval path was
tried end to end before the handover. Three different people created cases.

## The final version reached every status

| Status | Cases |
|---|---|
| Draft | 23 |
| Pending 1st Level Approval | 7 |
| Approved | 1 |
| Rejected | 4 |
| Disposed | 8 |
| (blank, early tests) | 3 |

So the parts this copy of the source is missing were finished in SLB's environment:

- **Approve and reject change the case.** Two columns, `1stApprovalStatus` and `2ndApprovalStatus`, record
  each approver's decision as true or false, and the case status moves with them.
- **Rejection at either level works.** Cases were rejected at 1st level (`false`, blank) and at 2nd level
  (`true`, `false`).
- **Cases were closed as Disposed**, from 29 July 2025 onwards.
- **`Year` and `UID` columns were added mid-build.** `Year` first appears on case 41 (19 July) and `UID`
  on case 49 (22 July). The UID is a timestamp string such as `20250722202105388`, used to match records
  created in the same session before SharePoint assigns an ID.

## What was most often disposed

| Item | Rows |
|---|---|
| Laptop | 33 |
| Laptop cable lock | 20 |
| Desktop | 14 |
| Keyboard, Tablet | 5 each |
| Cable, Mouse | 4 each |
| Monitor, Laptop power adapter | 3 each |

Dell, Apple and HP make up most of the brands.

## Data quality problems

These are the things a real audit would trip over. They come from testing as much as real use, but each
one is something the app allowed, so each one is fixed in the v2 design.

- **Text where numbers belong.** `caseNo` on an item is text, so one item points at a case called
  `urmmmm`, and two items point at case 3, which no longer exists. Three items have no case at all.
- **Quantity is text.** Assets store `-` instead of an empty value, so quantities can't be summed directly.
- **True, true, True.** Approval columns hold `true`, `True`, `false` and `False` as text, not yes/no.
- **Status and approvals can disagree.** One case is *Disposed* with both approval columns blank, and one
  is *Approved* with no 1st level decision recorded.
- **Free-typed brands.** `DELL` and `Dell`, `HP` and `hp`, plus test values such as `424` and `jp`.
- **Approver fields mixed up.** On five cases the approver's email sits in the name column and a
  placeholder note sits in the email column.
- **Location list gaps.** 14 of 20 bases have no name, one base ID appears twice, a `PH` base is filed
  under Brunei, and the country is spelled both `Malaysia` and `MALAYSIA`.
- **Empty cases.** 15 cases have no items, mostly drafts abandoned while testing.
- **No history.** Each case keeps only its current status. When and by whom it moved from one status to
  the next is only in SharePoint's version history, not in the app.

## The supporting lists

The case, item and location lists tell most of the story, but the end-of-internship export also has the
smaller lists the app fed. These show how the closing steps and the side features were really used.

| List | Rows | What it shows |
|---|---|---|
| QUEST Number | 6 | Safety report numbers, on 6 cases |
| IBase Number | 5 | Manpower request numbers, on 4 cases |
| Watchlists | 13 | 9 cases shared, 22 July to 5 August |
| Contact Support Info | 9 | 4 inquiries, 2 bug reports, 3 "others" |
| IT Asset Lists | 482 | The device catalogue behind the dropdowns |
| Admins | 4 | Including a shared team mailbox |
| Requestor | 3 | Everyone who opened the app, recorded at login |

### Closing a case didn't check the paperwork

The process says nothing leaves the base until a QUEST safety report and IBase manpower requests are
raised. The app recorded both, but didn't require them before *Item Disposed?*:

| Disposed cases | Count |
|---|---|
| With both QUEST and IBase numbers | 3 |
| With only one of them | 3 |
| With neither | 2 |

So 5 of the 8 disposed cases were closed with paperwork missing. In testing that's harmless, but in real use
it's the gap an auditor would find first. The fix is simple: disable *Item Disposed?* until both numbers
exist.

### One case, several IBase requests

One case carries two IBase numbers. That matches the manual process: one manpower request to carry the
items to the junkyard and another to destroy the hard drives. Keeping IBase in its own list, rather than as
a column on the case, was the right call, because it lets a case hold as many requests as it needs.

### No format checks on reference numbers

QUEST numbers range from 10 to 14 digits, and one is the text `jajajajaja`. IBase numbers range from 7 to 13
digits, one with a leading zero. Both are stored as text with no pattern check, so any typo is accepted.

### The watchlist was used, mostly for testing

13 watchlist entries cover 9 cases. Three different people were added, but 9 of the 13 entries are the
builder adding himself while testing. The feature works, but there isn't enough real use to say whether
teams wanted it.

### Contact Us got real requests

Most of the 9 messages are test text, but two are genuine requests to add a new base. That fits the design:
requestors can't add bases themselves, so Contact Us is how a missing location reaches an admin.

### The catalogue came from the asset register

The IT Asset Lists catalogue has 482 models across 13 item types: 430 assets and 52 non-assets. Laptops
(219) and desktops (197) make up most of it, and Dell alone accounts for about 60%. It looks imported
from the wider asset register rather than built for disposal. It includes server and industrial brands
(Supermicro, Rockwell, VMware), and the manufacturer column has `DELL` and `Dell`, plus `Unknown`, `CLONE`
and `-`. A disposal-only catalogue would be shorter and easier to pick from.

## How v2 answers these

[it-asset-disposal-app-v2](https://github.com/7YMERA/it-asset-disposal-app-v2) is the redesign. Its data
model uses real types, foreign keys and a status history table, which rules out most of the list above
by design.
