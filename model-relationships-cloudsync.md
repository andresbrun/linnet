# Model Relationships and CloudSync

This document explains how `Person` and `Event` relate, and how CloudKit sync is wired through SwiftData.

## Data Model Relationship

The domain is many-to-many between people and events.

Implementation detail: relationships are mirrored through UUID arrays (not native SwiftData relationship properties):

- `EventEntity.personIDsData` stores `[UUID]` as JSON.
- `PersonEntity.eventIDsData` stores `[UUID]` as JSON.

```mermaid
erDiagram
    PERSON {
      UUID id
      string displayName
      string nickname
      string notes
      string contactIdentifier
      date birthday
      data photoData
      UUID[] eventIDs
      date createdAt
      date updatedAt
    }

    EVENT {
      UUID id
      string title
      string notes
      date startDate
      int recurrenceValue
      string recurrenceUnit
      string colorThemeRaw
      UUID[] personIDs
      bool notificationEnabled
      int notificationOffsetValue
      string notificationOffsetUnit
      data reminderStepsData
      date createdAt
      date updatedAt
    }

    PERSON }o--o{ EVENT : "linked by mirrored UUID arrays"
```

## Consistency Rules

`PersistenceClient.live` keeps both sides aligned:

- Saving an event updates affected `PersonEntity.eventIDs`.
- Saving a person updates affected `EventEntity.personIDs`.
- Deleting either side removes stale IDs from the opposite side.

This guarantees bidirectional consistency even without explicit ORM relationships.

## CloudSync Design

`SharedContainer.makeModelContainer()` tries CloudKit-backed storage first, then falls back to local-only storage if initialization fails.

```mermaid
flowchart TD
    A["App/Extension starts"] --> B["SharedContainer.makeModelContainer()"]
    B --> C{"Default ModelContainer succeeds?"}
    C -- Yes --> D["CloudKit enabled"]
    C -- No --> E["Create local-only ModelConfiguration(cloudKitDatabase: .none)"]
    E --> F{"Local-only succeeds?"}
    F -- Yes --> G["Run in local-only mode + keep init error diagnostics"]
    F -- No --> H["Fail startup path (throw)"]
```

## Sync and Diagnostics Flow

`PersistenceClient` exposes:

- `observeRemoteChanges()` via:
  - `ModelContext.didSave`
  - `.NSPersistentStoreRemoteChange`
  - `NSPersistentCloudKitContainer.eventChangedNotification`
- `forceSync()` (save + fetch roundtrip)
- `getDiagnostics()` (mode, store URL, iCloud account status, event count, init error)

```mermaid
sequenceDiagram
    participant UI as SettingsFeature
    participant PC as PersistenceClient
    participant SC as SharedContainer
    participant CK as CKContainer
    participant NC as NotificationCenter

    UI->>PC: forceSync()
    PC->>PC: context.save() + fetches
    PC-->>UI: success/failure

    UI->>PC: getDiagnostics()
    PC->>SC: read isCloudKitEnabled/initError
    PC->>CK: accountStatus()
    PC-->>UI: CloudSyncDiagnostics

    NC-->>PC: didSave / remote change / CloudKit eventChanged
    PC-->>UI: AsyncStream<Void> update signal
```

