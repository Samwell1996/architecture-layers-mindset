# 📘 Documentation Overview

This folder contains all architecture documentation for the project.
It describes the core principles, layer boundaries, and patterns used across the application.

The goal is to provide a **LEGO-like construction system** for React Native applications:\*\*
modular, composable, scalable, and easy to reason about.

---

## 📂 Documents

### **1. `hybrid-ddd-sliced-overview.md`**

High-level architecture overview combining:

- DDD domain structure
- Feature-Sliced UI organization
- Normalized reactive MobX store
- Shared global layer
- Folder structure & boundaries
- Purpose and philosophy

This is the main conceptual document.

---

### **2. `mobx-store-architecture.md`**

Deep dive into the normalized MobX entity store:

- entities
- schemas
- models
- collections
- TTL caching
- async ducks
- reactivity model
- example flows and usage

This defines the **state engine** powering the entire app.

---

### **3. `copilot-system-core.md`**

Rules for automated architecture enforcement:

- allowed and forbidden patterns
- store/UI responsibilities
- correct usage of ducks, selectors, effects
- code rewrite rules

This enables AI tools (Copilot / ChatGPT) to write code that strictly follows the architecture.

---

### **4. `copilot-system-app.md`**

Application-wide AI system that defines how Copilot and ChatGPT should behave across all layers of the project:

- per-layer Copilot profiles (shared, ui, domain, api, global)
- enforced boundaries between layers
- naming and folder-structure rules
- architectural constraints for every module
- unified development workflow for all AI assistants

This file describes how the entire Copilot system works together to keep the project consistent, predictable, and fully aligned with the architecture - effectively turning the app into a self-enforcing, AI-driven LEGO construction system.

---

## 📐 Architecture Goals

- Clean separation of UI / Domain / Infrastructure
- Predictable and reactive state flow
- Scalable domain logic
- Minimal boilerplate
- Fast feature delivery
- High readability and maintainability
- Consistent shared primitives
- Reusable UI blocks
- Clear import boundaries

---

## 🔗 How to Navigate

Start here:
👉 `hybrid-ddd-sliced-overview.md`

Then go deeper:
👉 `mobx-store-architecture.md`

enforce patterns(store patterns):
👉 `copilot-system-core.md`

enforce patterns(app patterns):
👉 `copilot-system-app.md`

---

## ✔ Audience

This documentation is intended for:

- Senior engineers
- Technical leads
- Architects
- Anyone onboarding into the project
- AI assistants writing code under constraints

---

## 🏁 Summary

The documents in this folder form the **architectural foundation** of the entire application.
Every screen, feature, store, or API module should follow these principles to ensure long-term scalability and consistency.
