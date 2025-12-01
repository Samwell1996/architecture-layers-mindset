# Copilot System — Application-Wide AI Development Standards

This document defines how AI tools (Copilot / ChatGPT) must behave across the entire project.  
Each project layer has its own dedicated Copilot profile, and this system describes how they are combined into a unified, architecture-enforcing development workflow.

---

## 1. Purpose

The goal of this Copilot system is to:

- provide **AI awareness** of the project architecture  
- enforce clean boundaries automatically  
- prevent architectural drift  
- make onboarding extremely fast  
- turn the entire app into a **LEGO-like construction system** powered by consistent code generation  

---

## 2. Folder Structure for Copilot Profiles

```
.copilot/
 ├── shared.md        ← rules for shared primitives, utils, hooks
 ├── ui.md            ← rules for screens/features/widgets/entities
 ├── domain.md        ← rules for DDD stores, models, schemas
 ├── api.md           ← rules for API and service layer
 └── global.md        ← rules that apply everywhere
```

---

## 3. How the Profiles Work Together

### ✔ global.md  
Defines universal architectural rules:

- UI must not contain business logic  
- Domain must not import UI  
- API layer must not know domain logic  
- No Redux, React Query, Zustand  
- MobX normalized engine is mandatory  
- Ducks manage async state  
- Effects must call `.run()` only  
- Domain models must be reactive  
- Shared layer is globally accessible  

---

### ✔ shared.md  
Tells AI how to:

- generate UI primitives (`AppText`, `AppImage`, etc.)  
- format pure helpers  
- write reusable hooks  
- avoid domain or UI dependencies  

---

### ✔ ui.md  
Enforces UI-layer rules:

- screens/widgets/features only  
- no domain mutations  
- selectors only  
- ducks only via `.run()`  
- no DTO usage  
- UI-level entities are adapters  

---

### ✔ domain.md  
Enforces DDD rules:

- schemas  
- models  
- stores  
- collections  
- normalization  
- async ducks  
- TTL policies  

No React, no hooks, no components.

---

### ✔ api.md  
Defines:

- HTTP layer  
- ApiManager rules  
- error mapping  
- retries  
- transport  
- DTO to domain mapping  

---

## 4. Why This System Works

It turns AI tools into:

- junior developers that always obey architecture  
- senior developers that generate consistent patterns  
- reviewers that rewrite incorrect code automatically  

This gives the project:

- predictable code  
- zero architectural drift  
- clean boundaries  
- fast feature development  
- onboarding in minutes  
- future-proof codebase  

---

## 5. Activation in GitHub/IDE

Each file is loaded into GitHub Copilot Workspace context using:

- Copilot Custom Instructions  
- `.copilot` folder autoload  
- Workspace-level prompts  
- ChatGPT custom project profiles  

This makes AI *architecturally aware*.

---

## 6. Result

Your app becomes:

> **A self-enforcing architecture powered by AI.  
Developers write code — the system keeps it clean.**
