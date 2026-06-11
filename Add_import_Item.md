# Demo Flow: Import Items, Add Item & Accept Original Value

This document maps the three features for today's demo to actual backend code.

---

## 1. Import Items (Launch from Cotality / Streamline)

When the adjuster opens the Artigem "Contents" plugin from inside Cotality/Symbility, the embedded
launch automatically pulls the estimate items, builds the rooms (from diagrams), and creates the
items in Artigem.

```mermaid
flowchart TD
    A["Adjuster clicks 'Contents Workspace'\ninside Cotality (Symbility)"] --> B["GET /api/web/coreLogic/token\nWebCorelogicCommunicationController:747"]
    B --> C["getAccessTokenFromClaimsConnect()\nWebCoreLogicCommunicationServiceImpl:235\n(OAuth2 client_credentials -> Symbility)"]
    C --> D["webCorelogicService.authenticate(authCode...)\n-> CoreLogicCreateClaimDTO"]
    D --> E["fetchTokenFromCCandClaimDetails()\nWebCoreLogicCommunicationServiceImpl:2864"]
    E --> F["getTokenAndClaimIdentifiers()\n:2877"]
    F --> G["createClaimForEmbeddedIntegration()\n:3035\n(creates/links Claim+Policy in Artigem)"]
    G --> H["importEstimateItems(claimNumber, estimateId, token, ...)\n:3433"]

    H --> I["GET .../claims/{claimId}/estimates/{estimateId}\n(Symbility REST API)"]
    I --> J["Loop over EstimateItems[]\nbuild ClaimItemDetailDTO per item\n(coverage, subcoverage, diagramId,\ndiagramObjectId, actionCode, RCV/ACV, etc.)"]

    J --> K{"distinct diagramIds\nfound?"}
    K -- "yes" --> L["importDiagramsFromCotalityAndCreateRoomsInStreamline()\n:3989\nfor each diagramId"]
    L --> M["GET .../claims/{claimId}/diagrams/{diagramId}\n(fetch room/diagram objects)"]
    M --> N["Load existing CorelogicItemRoomDiagramMapping\nfor this claim (dedup check)\n:4011-4013"]
    N --> O{"Room already\nmapped for this\nDiagramObjectID?"}
    O -- "no" --> P["Create new Room\n+ CorelogicItemRoomDiagramMapping"]
    O -- "yes" --> Q["Skip - reuse existing Room"]
    P --> R
    Q --> R["Rooms ready"]

    K -- "no" --> R
    R --> S["webInventoryServiceImpl.addOrUpdateItemFromCoreLogic()\nWebInventoryServiceImpl:6797"]
    S --> T{"Items already\nexist for this\nclaim+estimate?"}
    T -- "none exist" --> U["addPostlossItems(claimItemDetailDTOArray, ...)\nWebInventoryServiceImpl:793\n-> creates Item rows, links to Room,\nclaim, policy, status, tags"]
    T -- "some new items" --> U
    T -- "all already imported" --> V["UpdateClaimConnectFields()\n(currently commented out -\nno-op for now)"]
```

**Talking points (keep it high-level):**
- Trigger: opening the Artigem plugin from within Cotality automatically calls our `/token` endpoint.
- We exchange tokens, identify/create the claim, then call `importEstimateItems()`.
- That method calls Symbility's Estimate API to get the line items (`EstimateItems[]`).
- Each item carries a `DiagramID` + `DiagramObjectID` — Symbility's way of saying "this item belongs to this room on this floor plan."
- Before creating items, we make sure the **Rooms** exist in Artigem (`importDiagramsFromCotalityAndCreateRoomsInStreamline`), using `CorelogicItemRoomDiagramMapping` as the bridge table so we never create the same room twice.
- Finally `addOrUpdateItemFromCoreLogic()` decides: brand-new import → create items via `addPostlossItems()`; already imported → (currently a no-op, update logic is commented out — this is part of "Update Estimate" which Gaurav covers tomorrow).

---

## 2. Add Item (Manual, from Artigem UI)

```mermaid
flowchart TD
    A["Adjuster clicks 'Add Item'\nin Artigem Contents UI"] --> B["POST /api/web/inventory/...\n(Add Item endpoint)\nWebInventoryController"]
    B --> C["addPostlossItems(itemsDTO[], files, fileDetails, isVendor)\nWebInventoryServiceImpl:793"]
    C --> D["Resolve Claim + Policy"]
    D --> E["Determine next Item Number\n(last item number + 1)"]
    E --> F["Load CorelogicItemRoomDiagramMapping\nfor claim (room context)\n:824-826"]
    F --> G["For each item:\n- create Item entity\n- set ApplyTax, TaxRate\n- set IsPostLoss = true\n- link Claim, Policy, Room\n- set ItemStatus = CREATED\n- set ItemTag (if any)\n- generate ItemUID"]
    G --> H["Save Item(s)\n+ attach media files (if any)"]
    H --> I["Return List<ClaimItemDetailDTO>\nto UI - new item appears\nin contents grid"]
```

**Talking points:**
- Same `addPostlossItems()` method is the workhorse for both the Cotality import AND manual "Add Item" — the difference is just *who* calls it and what's pre-filled.
- When added manually, the adjuster fills in description/category/etc. via the UI; when imported from Cotality, those fields come pre-populated from `EstimateItems[]`.
- Each new item gets a unique `ItemUID`, an auto-status of `CREATED`, and is linked to the claim/policy/room.

---

## 3. Accept Original Value (Item Replacement)

This feature lets the adjuster say: "the price Cotality/the system suggested for replacement is fine — just use the item's original cost as the replacement cost," without going through the full vendor/catalog comparison flow.

```mermaid
flowchart TD
    A["Adjuster clicks 'Accept Original Value'\non an item"] --> B["POST /api/web/inventory/accept/original/item/replacement\nWebInventoryController:1218"]
    B --> C["prepareCustomOriginalCostReplacementItem(itemId, claimId)\nWebInternalServiceImpl:1374"]
    C --> D["Load Item by itemId"]
    D --> E["Build CustomItemCatalogDTO:\n- customItemFlag = true\n- isReplacementItem = true\n- description = item.description\n- quantity = item.quantity\n- replacementCost = item.itemprice (ORIGINAL cost)\n- itemStatus = 'Comparable'"]
    E --> F["customItemComparable(customItemCatalogDTO, null, null)\nWebInternalServiceImpl:464\n-> saves replacement item record\n+ item-replacement table\n+ syncs to vendor schema (internal call)"]
    F --> G{"Success?"}
    G -- "yes" --> H["200 OK -\nItem now has a replacement\nentry using its OWN price\nas the replacement cost"]
    G -- "no" --> I["400 Bad Request -\n'While Accepting Original Cost\nsomething went wrong'"]
```

**Talking points:**
- "Accept Original Value" = treat the item's **own current price** (`item.getItemprice()`) as its replacement cost — no need to search a catalog or vendor for a comparable item.
- It builds a `CustomItemCatalogDTO` marked `isReplacementItem = true`, `itemStatus = "Comparable"`, with `replacementCost` set to the item's existing price.
- `customItemComparable()` then persists this as the item's replacement record and pushes it to the vendor schema via an internal call — so downstream (estimate/RCV-ACV calculations) treat this item as "resolved" without further vendor lookup.

---

## Quick Reference — File:Line

| Feature | Entry Point | Key Method |
|---|---|---|
| Import Items (launch) | `WebCorelogicCommunicationController.java:747` (`/token`) | `importEstimateItems()` — `WebCoreLogicCommunicationServiceImpl.java:3433` |
| Room/Diagram sync | (called from above) | `importDiagramsFromCotalityAndCreateRoomsInStreamline()` — `:3989` |
| Item creation/sync | (called from above) | `addOrUpdateItemFromCoreLogic()` — `WebInventoryServiceImpl.java:6797` |
| Add Item (manual) | `WebInventoryController` (Add Item endpoint) | `addPostlossItems()` — `WebInventoryServiceImpl.java:793` |
| Accept Original Value | `WebInventoryController.java:1218` (`/accept/original/item/replacement`) | `prepareCustomOriginalCostReplacementItem()` — `WebInternalServiceImpl.java:1374` |
