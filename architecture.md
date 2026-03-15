# Linnet Architecture

This document describes the current high-level architecture of the app and its main runtime flows.

## Overview

Linnet is structured around:

- `ComposableArchitecture` reducers/features for app behavior.
- Dependency-injected clients for persistence, notifications, analytics, and suggestions.
- `SwiftData` as the persistence layer, with CloudKit-backed sync when available.
- A Share Extension that writes events into the same data model/container.

```mermaid
flowchart TB
    subgraph App["Linnet App"]
      A["LinnetApp"] --> B["AppFeature"]
      B --> C["EventListFeature"]
      B --> D["PeopleFeature"]
      B --> E["SettingsFeature"]
      B --> F["OnboardingFeature"]
    end

    subgraph Dependencies["Dependency Clients"]
      P["PersistenceClient"]
      N["NotificationClient"]
      AN["AnalyticsClient"]
      S["EventSuggestionClient"]
    end

    subgraph Domain["Domain Services / Models"]
      RE["RecurrenceEngine"]
      ES["EventSuggestionService"]
      M["Event + Person"]
    end

    subgraph Data["Storage"]
      SD["SwiftData (ModelContainer)"]
      CK["CloudKit (when available)"]
      LO["Local-only fallback"]
    end

    subgraph ShareExt["Share Extension"]
      SV["ShareViewController"] --> SF["ShareFormView"]
      SF --> SD
    end

    C --> P
    C --> N
    C --> AN

    D --> P
    D --> S
    D --> AN

    E --> P
    E --> N
    E --> AN

    B --> P
    B --> N
    B --> AN
    B --> RE

    S --> ES
    ES --> M
    P --> M
    P --> SD
    SD --> CK
    SD --> LO
```

## Startup Flow

1. `LinnetApp` builds a `PersistenceClient`.
2. If launched with screenshot flags, it uses mocked data.
3. Otherwise, it creates a `ModelContainer` through `SharedContainer.makeModelContainer()`.
4. The store is created with `AppFeature` and injected dependencies.
5. On app launch action, notifications are re-scheduled for enabled events and analytics are logged.

## Data and Extension Integration

- App and Share Extension both rely on the same schema (`EventEntity`, `PersonEntity`) via `SharedContainer`.
- The extension can create events directly, which then become visible in the main app.
