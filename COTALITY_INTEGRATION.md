# Cotality Integration – Understanding Document

## 1. What is Cotality?

Cotality (formerly known as CoreLogic) is a **third-party insurance claims platform** whose API is accessed via the **Symbility ClaimsConnect** REST API (hosted at `*.symbility.net`). In the context of ArtigemRS, it is the external insurance company partner that sends claim data, floor plan diagrams, and estimate items into the Streamline system.

Cotality is registered in the system as a company with the following identifiers:

| Property           | Value                          |
|--------------------|--------------------------------|
| Base URL           | `http://45.79.5.228:8080/cotality` |
| Schema             | `COTALITYONE`                  |
| Registration Number| `COTALITYONE`                  |

**Config file:** [company_configuration.xml](ArtigemRS-FI/src/main/resources/configuration/company_configuration.xml) (lines 46–50)

---

## 2. High-Level Integration Flow

```
Cotality/Symbility Platform
        │
        │  [ClaimsConnect webhook / embedded session trigger]
        ▼
fetchTokenFromCCandClaimDetails()          ← Entry point
        │
        ▼
getTokenAndClaimIdentifiers()              ← Decrypts claim hash → gets claimNumber
        │
        ▼
importEstimateItems()                      ← Calls Symbility REST API to get estimate items
        │
        ├──► importDiagramsFromCotalityAndCreateRoomsInStreamline()   [per unique DiagramID]
        │         │
        │         └──► Calls Symbility diagrams API → Creates Room + CorelogicItemRoomDiagramMapping
        │
        └──► addOrUpdateItemFromCoreLogic()   ← Saves estimate line items as inventory Items
```

---

## 3. Method Reference

### 3.1 Entry Point — `fetchTokenFromCCandClaimDetails()`

**File:** [WebCoreLogicCommunicationServiceImpl.java](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L2862)

```java
public CorelogicTokenDTO fetchTokenFromCCandClaimDetails(
    CoreLogicCreateClaimDTO coreLogicCreateClaimDTO,
    HttpServletRequest request,
    String token,
    String environment
)
```

- Public entry point called from the controller when Cotality triggers the embedded session.
- Delegates to `getTokenAndClaimIdentifiers()`.

---

### 3.2 `getTokenAndClaimIdentifiers()`

**File:** [WebCoreLogicCommunicationServiceImpl.java](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L2876)

```java
private CorelogicTokenDTO getTokenAndClaimIdentifiers(
    CoreLogicCreateClaimDTO coreLogicCreateClaimDTO,
    HttpServletRequest request,
    String token,
    String environment
)
```

- Calls `createClaimForEmbeddedIntegration()` to set up the claim in Streamline.
- Decrypts the `claimHash` to extract the `claimNumber`.
- Calls `importEstimateItems()` with the `estimateId`, `token`, and `partnerId` from `CoreLogicCreateClaimDTO`.
- Returns `CorelogicTokenDTO` back to the client for the embedded session redirect.

---

### 3.3 `importEstimateItems()` — Core Orchestrator

**File:** [WebCoreLogicCommunicationServiceImpl.java](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L3432)

```java
@Transactional
public Boolean importEstimateItems(
    String claimNumber,
    String estimateId,
    String authToken,
    String partnerId,
    String environment
)
```

**What it does:**

1. Looks up the `Claim` entity by `claimNumber`.
2. Calls the **Symbility Estimates REST API**:
   ```
   GET https://{environment}.symbility.net/rest-api/v03_15/claims/{claimId}/estimates/{estimateId}
   Authorization: Bearer {authToken}
   Partner-Request-ID: {partnerId}
   ```
3. Parses the `EstimateItems` JSON array from the response.
4. For each `EstimateItem`, populates a `ClaimItemDetailDTO` with fields:
   - `EstimateItemID`, `Quantity`, `Description`, `PurchaseDate`, `PurchaseCost`
   - `DiagramID`, `DiagramObjectID`, `ClaimCoverageID`, `ClaimSubcoverageID`, `ActionCode`
5. Collects a `Set<Integer> distinctDiagramIds` — unique diagram IDs across all estimate items.
6. For each unique `DiagramID`, calls `importDiagramsFromCotalityAndCreateRoomsInStreamline()`.
7. Calls `webInventoryServiceImpl.addOrUpdateItemFromCoreLogic()` to persist the items.

---

### 3.4 `importDiagramsFromCotalityAndCreateRoomsInStreamline()` — Room Sync

**File:** [WebCoreLogicCommunicationServiceImpl.java](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L3988)

```java
@Transactional
public void importDiagramsFromCotalityAndCreateRoomsInStreamline(
    String environment,
    Claim claim,
    Integer diagramId,
    String authToken
)
```

**What it does:**

1. Calls the **Symbility Diagrams REST API**:
   ```
   GET https://{environment}.symbility.net/rest-api/v03_15/claims/{claimId}/diagrams/{diagramId}
   Authorization: Bearer {authToken}
   Partner-Request-ID: {partnerRequestId}
   ```
2. Parses the `DiagramObjects` array from the JSON response.
3. Loads all existing `CorelogicItemRoomDiagramMapping` records for the claim (to avoid duplicates).
4. For each `DiagramObject` where `Type == "Room"`:
   - Extracts `DiagramObjectID` and `Name`.
   - Checks if a mapping already exists (same `diagramId` + `diagramObjectId`).
   - If **no existing mapping**:
     - Creates a new `Room` entity with `roomType = 15` (hardcoded default type), assigns it to the claim and policy.
     - Creates a `CorelogicItemRoomDiagramMapping` linking the new room to the `diagramId` + `diagramObjectId`.
     - Saves both to the database.

---

### 3.5 `addOrUpdateItemFromCoreLogic()`

**File:** [WebInventoryServiceImpl.java](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebInventoryServiceImpl.java#L6797)

```java
public void addOrUpdateItemFromCoreLogic(
    List<ClaimItemDetailDTO> claimItemDetailDTOList,
    List<Integer> estimateIds,
    String claimnumber
)
```

**What it does:**

- Checks whether items with the given `estimateIds` already exist in the database.
- If **none exist**: calls `addPostlossItems()` to insert all items as new inventory.
- If **some exist but counts differ**: determines which items are new and calls `addPostlossItems()` for only the new ones.
- If **all exist**: skips (no-op).

---

### 3.6 Helper Methods (Diagram Mapping Support)

#### `getCorelogicDiagramObjectIdsofAllRoomsofClaim()`
[WebCoreLogicCommunicationServiceImpl.java:3774](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L3774)

- Returns all `CorelogicItemRoomDiagramMapping` records for rooms that belong to a given claim (using subquery).
- Used when sending estimate updates back to Cotality.

#### `getDiagramIdForRoom()`
[WebCoreLogicCommunicationServiceImpl.java:3790](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L3790)

- Given a list of `CorelogicItemRoomDiagramMapping` and a `Room`, returns a `Map` containing `diagramId` and `diagramObjectId` for that room.
- Used when building estimate update payloads to send back to Symbility.

#### `getRoomListByClaim()`
[WebCoreLogicCommunicationServiceImpl.java:3690](ArtigemRS-FI/src/main/java/com/pims/web/service/impl/WebCoreLogicCommunicationServiceImpl.java#L3690)

- Queries all `Room` entities for a given claim via Criteria API.

---

## 4. API Endpoints Used

| Purpose | Method | URL Pattern |
|---|---|---|
| Fetch estimate items | `GET` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/estimates/{estimateId}` |
| Fetch diagram / room layout | `GET` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/diagrams/{diagramId}` |
| Update estimate | `POST` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/estimates/{estimateId}/update` |
| Fetch all estimates for assignment | `GET` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/estimates?assignmentID={assignmentId}` |
| Upload documents | `POST` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/documents` |
| Update custom fields | `POST` | `https://{env}.symbility.net/rest-api/v03_15/claims/{claimId}/partner-requests/{partnerRequestId}/update-status` |

**Auth headers on every request:**
```
Authorization: Bearer {authToken}
Partner-Request-ID: {partnerRequestId}
```

**Environment variable:** `corelogic.Oauth.environment` in [application.properties](ArtigemRS-FI/src/main/resources/application.properties) (default: `staging`)

---

## 5. DTO — `ClaimItemDetailDTO`

**File:** [ClaimItemDetailDTO.java](ArtigemRS-FI/src/main/java/com/pims/dto/ClaimItemDetailDTO.java)

This DTO is the central transport object carrying one estimate line item from Cotality into Streamline. Key fields relevant to Cotality:

| Field | Type | Source (Symbility JSON) | Purpose |
|---|---|---|---|
| `claimsConnectEstimateId` | `Integer` | `EstimateItemID` | Links item to Cotality estimate |
| `diagramId` | `Integer` | `DiagramID` | Which floor plan diagram the item belongs to |
| `diagramObjectId` | `Integer` | `DiagramObjectID` | Which room/object within that diagram |
| `coverage` | `Integer` | `ClaimCoverageID` | Cotality coverage ID |
| `subcoverage` | `Integer` | `ClaimSubcoverageID` | Cotality sub-coverage ID |
| `actionCode` | `String` | `ActionCode` | Cotality action code for the item |
| `description` | `String` | `EstimateItemContents.Description` | Item description |
| `dateOfPurchase` | `String` | `EstimateItemContents.PurchaseDate` | Purchase date |
| `totalStatedAmount` | `Double` | `EstimateItemContents.PurchaseCost` | Total cost |
| `quantity` | `Integer` | `Quantity` | Quantity |
| `insuredPrice` | `Double` | Computed: `totalStatedAmount / quantity` | Per-unit price |
| `claimId` | `Long` | Resolved from local DB | Streamline claim ID |
| `claimNumber` | `String` | Resolved from local DB | Streamline claim number |
| `applyTax` | `Boolean` | Hardcoded `true` | Tax flag |

---

## 6. Models (JPA Entities)

### 6.1 `Room`

**File:** [Room.java](ArtigemRS-FI/src/main/java/com/pims/model/Room.java)  
**DB Table:** `room`

Represents a physical room within a claim, populated from Cotality diagram data.

| Column | Type | Description |
|---|---|---|
| `ID` | `Long` | Primary key |
| `NAME` | `String` | Room name from diagram (`DiagramObject.Name`) |
| `CREATE_DATE` | `Date` | Set to `new Date()` on import |
| `POLICY_ID` | FK → `Policy` | The policy the room belongs to |
| `CLAIM` | FK → `Claim` | The claim the room belongs to |
| `ROOM_TYPE_ID` | FK → `RoomType` | Always set to `RoomType ID = 15` on Cotality import |
| `MODIFIED_DATE` | `Date` | Last modification date |
| `MODIFIER_ID` | `Long` | Who modified it |

Relationships:
- `@OneToMany` → `Item` (items in this room)
- `@OneToMany` → `Note`
- `@OneToMany` → `MediaFile`
- `@ManyToOne` → `Claim` via `CLAIM` column
- `@ManyToOne` → `Policy` via `POLICY_ID`

---

### 6.2 `CorelogicItemRoomDiagramMapping`

**File:** [CorelogicItemRoomDiagramMapping.java](ArtigemRS-FI/src/main/java/com/pims/model/CorelogicItemRoomDiagramMapping.java)  
**DB Table:** `corelogic_item_room_diagram_map`

This is the **bridge table** linking Cotality's diagram coordinate system to Streamline's Room entities.

| Column | Type | Description |
|---|---|---|
| `ID` | `Long` | Primary key |
| `DIAGRAM_ID` | `Integer` | `DiagramID` from Symbility API (which floor plan) |
| `DIAGRAM_OBJECT_ID` | `Integer` | `DiagramObjectID` from Symbility API (which room shape) |
| `ROOM_ID` | FK → `Room` | The local Streamline `Room` this maps to |

This table allows the system to:
1. Avoid creating duplicate rooms on re-import (de-duplication check).
2. Reverse-lookup the Cotality diagram coordinates when sending estimate updates back to Symbility.

---

### 6.3 `CorelogicItemNoteRoomMapping`

**File:** [CorelogicItemNoteRoomMapping.java](ArtigemRS-FI/src/main/java/com/pims/model/CorelogicItemNoteRoomMapping.java)  
**DB Table:** `corelogic_item_note_room_map`

Maps a Streamline `Item` to a Cotality `NoteID` on the Symbility side. Used when syncing item notes back to Cotality.

| Column | Type | Description |
|---|---|---|
| `ID` | `Long` | Primary key |
| `ITEM` | FK → `Item` | The local item |
| `NOTE_ID` | `Integer` | Cotality note ID on the Symbility side |

---

## 7. DAO Dependencies

The following DAOs are used in the Cotality flow inside `WebCoreLogicCommunicationServiceImpl`:

| DAO Field | Entity | Used For |
|---|---|---|
| `claimBaseDAO` | `Claim` | Look up claim by number; save `AssignmentId`, `DiagramId` |
| `roomBaseDAO` | `Room` | Save new rooms; query rooms by claim |
| `roomTypeBaseDAO` | `RoomType` | Fetch `RoomType ID=15` for new rooms |
| `corelogicItemRoomDiagramMappingBaseDAO` | `CorelogicItemRoomDiagramMapping` | Save/query diagram-to-room mappings |
| `corelogicItemNoteRoomMappingBaseDAO` | `CorelogicItemNoteRoomMapping` | Query/delete note-to-item mappings on sync |

---

## 8. Configuration

### `company_configuration.xml`
[ArtigemRS-FI/src/main/resources/configuration/company_configuration.xml](ArtigemRS-FI/src/main/resources/configuration/company_configuration.xml)

```xml
<company>
  <url>http://45.79.5.228:8080/cotality</url>
  <schema>COTALITYONE</schema>
  <registrationNumber>COTALITYONE</registrationNumber>
</company>
```

This registers Cotality as an Artigem tenant. The `schema` is the database schema name used when this company logs in, and `registrationNumber` is the company identifier.

### `application.properties`
[ArtigemRS-FI/src/main/resources/application.properties](ArtigemRS-FI/src/main/resources/application.properties)

```
corelogic.Oauth.environment = staging
```

Controls which Symbility environment (`staging`, `production`, etc.) is targeted for all API calls.

---

## 9. Data Flow — Step by Step

```
Step 1: Cotality sends embedded session request
  → CoreLogicCreateClaimDTO arrives at controller
  → Contains: estimateId, partnerRequestID, companyName, claimNumber

Step 2: fetchTokenFromCCandClaimDetails()
  → Creates/updates Claim in Streamline DB via createClaimForEmbeddedIntegration()
  → Returns claimHash (encrypted claimNumber)

Step 3: importEstimateItems(claimNumber, estimateId, authToken, partnerId, env)
  → GET https://{env}.symbility.net/.../estimates/{estimateId}
  → Response: { EstimateItems: [...], AssignmentID: X }
  → Stores AssignmentID on Claim

Step 4: Parse EstimateItems array
  → Each item → ClaimItemDetailDTO
    Fields populated: claimsConnectEstimateId, diagramId, diagramObjectId,
                      description, purchaseDate, purchaseCost, quantity,
                      coverage, subcoverage, actionCode

Step 5: For each unique DiagramID found in items:
  → importDiagramsFromCotalityAndCreateRoomsInStreamline(env, claim, diagramId, token)
  → GET https://{env}.symbility.net/.../diagrams/{diagramId}
  → Response: { DiagramObjects: [{ Type, DiagramObjectID, Name }, ...] }
  → Filter DiagramObjects where Type == "Room"
  → For each room object not already in corelogic_item_room_diagram_map:
      - INSERT INTO room (name, claim, policy, room_type=15, create_date)
      - INSERT INTO corelogic_item_room_diagram_map (diagram_id, diagram_object_id, room_id)

Step 6: addOrUpdateItemFromCoreLogic(dtoList, estimateIds, claimNumber)
  → Check if items with those estimateIds already exist in DB
  → If new: call addPostlossItems() to insert into `item` table
  → Items are now linked to rooms via diagramObjectId/diagramId matching
```

---

## 10. Key Relationships Summary

```
Symbility Claim ──────────────────────────── Streamline Claim
     │                                              │
     │  DiagramID + DiagramObjectID                 │
     ▼                                              ▼
CorelogicItemRoomDiagramMapping ◄──────────── Room (room table)
  (corelogic_item_room_diagram_map)
     │
     └── diagramId    = Cotality floor plan ID
     └── diagramObjectId = Cotality room shape ID
     └── room_id      = FK to local Room entity

Symbility EstimateItem ────────────────────── Item (item table)
     │                                              │
     └── EstimateItemID ──────────────────── claimsConnectEstimateItemId
     └── DiagramObjectID ─────────────────── matched via CorelogicItemRoomDiagramMapping → Room
```
