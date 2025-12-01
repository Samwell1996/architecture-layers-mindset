# Hybrid Architecture Overview — DDD + Feature-Sliced UI + Shared Core

## 🎯 Goal of the Architecture

The main goal of this architecture is to build a **LEGO‑like construction system** for React Native applications — where every layer is modular, replaceable and composable, enabling extremely fast feature development without sacrificing scalability, clarity or maintainability.

The application should feel like assembling blocks:

- UI blocks
- domain blocks
- shared primitives
- normalized reactive state blocks

All layers stay independent but composable.

---

# 1. High-Level Structure

```
src/
│
├── app/               ← navigation, providers, app entry
├── screens/           ← screen-level pages (RN Navigation screens)
├── modals/           ← modals-level pages (bottom sheet modals, RN Navigation ofc)
├── ui/                ← feature-sliced UI (features/widgets/entities)
├── stores/            ← DDD domain logic (models, schemas, store logic)
├── api/               ← DDD service layer (transport, endpoints)
├── shared/            ← global shared primitives/hooks/helpers/styles
└── assets/            ← images, animations, fonts
```

### Note on Modals

`modals/` is **not part of UI layer**, because modals in React Navigation behave like screens.
Therefore modals live next to screens:

```
src/screens/
src/modals/
```

---

# 2. Shared Layer (`src/shared`)

The Shared Layer contains globally reusable resources.
It is the **only layer that can be imported everywhere**.

```
src/shared/
 ├── components/       ← UI primitives (AppText, AppIcon, AppImage)
 ├── hooks/            ← shared hooks (keyboard, debounce, dimensions)
 ├── lib/              ← pure helpers (formatters, validators)
 ├── styles/           ← global styles, tokens, colors, typography
 └── config/           ← constants, env, app settings
```

---

# 3. UI Layer — Feature-Sliced (`src/ui`)

Responsible only for UI composition.
Contains **no business logic**.

```
src/ui/
 ├── features/         ← feature-level UI logic + domain binding
 ├── widgets/          ← reusable medium UI blocks
 └── entities/         ← mapping domain models → UI props
```

UI imports:

- ducks
- selectors
- shared primitives
- models (via mappers only)

---

# 4. Screens (`src/screens`)

Screens are the **presentation layer**.
They orchestrate UI blocks and initiate requests.

Rules:

- no business logic
- no domain mutations
- only layout, navigation and duck triggers

---

# 5. Domain Layer — DDD (`src/stores`)

Contains all intelligence of the app.

```
src/stores/
 ├── core/             ← normalization engine, EntitiesStore, collections
 ├── <DomainA>/        ← model.ts, schema.ts, store.ts
 ├── <DomainB>/        ← model.ts, schema.ts, store.ts
 └── index.ts          ← RootStore (dependency container)
```

Domain responsibilities:

- normalized reactive data graph
- entity relationships
- TTL caching
- async flows via ducks
- computed domain logic

---

# 6. API Layer — Service Boundary (`src/api`)

```
src/api/
 ├── manager.ts        ← ApiManager (TTL, retries, transport)
 ├── endpoints/        ← domain-separated HTTP methods (Posts/Chats, etc)
 └── types/            ← DTOs, params
```

Acts as the infrastructure boundary.

---

# 7. Data Flow

```
UI (screens/widgets/features)
   ↓ triggers
duck.run()

Domain Store (DDD)
   ↓ requests data
ApiManager.get()

API Layer
   ↓ fetch + normalization
EntitiesStore.merge()

Domain Models update
   ↓ reactive mapping
UI rerenders via observer()
```

Flow is unidirectional, reactive, normalized.

---

# 8. Architectural Principles

✔ UI is thin
✔ Domain is isolated
✔ State is normalized
✔ Shared layer is global
✔ Clear import boundaries
✔ Fully reactive models
✔ Predictable async flows

---

# 9. Benefits

- LEGO-like modular development
- Clean boundaries between layers
- Scalable domain logic
- Fast UI assembly
- Easy onboarding
- Reduced boilerplate
- High testability
- Perfect for large, long-lived apps
