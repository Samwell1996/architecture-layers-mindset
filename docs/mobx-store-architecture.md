# ⚡️ Entity Graph System powered by MobX

## 🧠 System Description

### 📌 What it is

The **Entity Graph System (EGS)** is a schema-driven, normalized data engine that manages entities, relationships, caching, and lifecycle in a fully deterministic and reactive way.

EGS operates on three interconnected graph layers:

1. **Domain Entity Graph**
   Derived from schemas and normalized entities.
   Defines parent/child relations, cardinality, and domain structure.

2. **GC Structural Graph**
   Built on demand for garbage collection (TTL, LRU, orphan detection).
   Provides reachability analysis and lifecycle enforcement for every entity.

3. **MobX Reactive Graph**
   MobX constructs a fine-grained dependency graph for reactivity.
   Components re-render only when the specific entities they depend on change.

All three layers share a **single source of truth — normalized entities**, ensuring consistency, predictability, and scalable performance.

### 📌 What it replaces

EGS removes the need for:

- Redux / Redux Toolkit
- RTK Query
- React Query
- manual normalization
- handwritten caching layers
- complex memoization
- ad-hoc relationship management

Instead, your domain, networking, caching, and reactivity all flow from:

- **Entity Schemas**
- **Graph Builder**
- **Garbage Collector**
- **MobX Reactive Engine**

### 📌 Why it exists

Traditional client-side state solutions treat data as flat “bags of state”.
EGS treats data as **interconnected entities inside a graph**, enabling:

- predictable GC
- automatic orphan cleanup
- structural TTL/LRU eviction
- consistent relationship mapping
- stable and clean domain models
- minimal re-renders due to MobX’s dependency graph

This system brings backend-level data engineering discipline to the client.

---

### 🕸️ How the Three Graphs Interact

EGS operates through three coordinated graphs, each serving a different responsibility layer:

1. **Domain Entity Graph**
   Built from schemas + normalized entities.
   Defines logical connections (author → viewer, event → participants).

2. **GC Structural Graph**
   Reconstructed from snapshots on demand.
   Used for TTL, LRU, and orphan detection.
   Independent from MobX.

3. **MobX Reactive Graph**
   Built automatically during model access.
   Determines which components re-render.

---

### ⚙️ Core Principles

| Principle                          | Description                                                                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------- |
| 🧩 **Entity-based data model**     | All normalized data lives inside `EntitiesStore` and is structured by `ENTITY_KEY`.           |
| 🕸️ **Three-layer Graph System**    | Domain Graph → GC Structural Graph → MobX Reactive Graph. Each layer serves a different role. |
| ♻️ **Garbage Collector (TTL/LRU)** | Automatic entity lifecycle management using reachability, TTL and LRU strategies.             |
| 🔗 **Entity Schemas**              | Declares relations (1:1 / 1:N / nested) and model classes for normalization.                  |
| 🧠 **Models**                      | DTO → Model transformation with computed fields and reactive graph links.                     |
| 📜 **Collections**                 | Reactive paginated lists with merge/append/prepend/reset operations.                          |
| ⚡️ **Async Ducks**                | Declarative async layer triggering data flow, normalization, merge, GC and persist.           |
| 💾 **Persist Layer**               | Versioned snapshot system: Extractor → Serializer → Processor → Store.                        |
| 🌍 **RootStore**                   | Dependency container: API, Entities, GC, Persist, domain stores.                              |

---

### 🧩 Key Components

#### 1️⃣ Entity Schema

Defines relationships between entities and which model to use for normalization.
It also declares how nested objects are flattened into `viewerId`, `viewersId` references.

```ts
export const postSchema = new EntitySchema(
	ENTITY_KEY.POST,
	{
		viewer: viewerSchema, // one-to-one
		viewers: [viewerSchema], // one-to-many
	},
	{ model: PostModel }
);
```

---

#### 2️⃣ Entity Collection

Represents a **normalized, reactive list** of entities.
It stores only IDs internally, while `getList` dynamically hydrates actual models from the entity cache.
Supports pagination, merging, `append`, `prepend`, and automatic MobX reactivity.

Collections expose a fully serializable snapshot (items, limit, hasNoMore, reversed, pageNumber), which is persisted and restored by the Persist Layer.
Only the snapshot is stored — models are re-hydrated dynamically via `EntitiesStore` on access.

---

#### 3️⃣ Model

Wraps raw DTO data with MobX observables and computed logic.
Each model knows how to access related entities through `getLinkedEntities(this)`, without strong references (so no memory leaks).

```ts
get author() {
  const entities = getLinkedEntities(this);
  return entities.getEntity(ENTITY_KEY.VIEWER, this.viewerId);
}
```

---

#### 4️⃣ Async Ducks

A wrapper around async functions that keeps their state observable:

```ts
{
	isLoading: boolean;
	isRetrying: boolean;
	error: Error | null;
	data: TResult | null;
	hasEverRun: boolean;
}
```

Each duck can have **independent states** (via Proxy → `fetchPosts["active"]`).

---

#### 5️⃣ RootStore

Central dependency container that connects:

- `root.api` — API layer
- `root.entities` — normalized cache
- `root.posts`, `root.viewer`, etc. — domain stores

All stores receive the same normalized reference graph.

---

#### 6️⃣ EntitiesStore

Global registry of all normalized entities.
When normalized data comes from the API, `EntityCollection` merges it here.
Every model can instantly read related entities without additional fetches.

---

#### 7️⃣ Data Flow Summary

```
normalize(data)
    ↓
EntitiesStore.merge()
    ↓
Persist Layer (snapshot → serialize → save)
    ↓
(optional) Garbage Collector (manual or scheduled run)
    ↓
MobX reactive graph
    ↓
UI updates

```

**Result:**
One-directional but fully reactive flow:
API updates → cache updates → UI reacts instantly.

---

#### 8️⃣ Design Goals

| Goal             | Description                                      |
| ---------------- | ------------------------------------------------ |
| 🧩 Scalability   | Add new entities with minimal setup.             |
| ⚡️ Reactivity   | Instant UI updates when data changes.            |
| 💾 Normalization | Shared references between all entities.          |
| 🧱 Structure     | Clear separation between async, model, and data. |
| 🧠 Simplicity    | Minimal API surface; one purpose per concept.    |

---

# 🧩 Entity Structure and Schemas

Each entity module (e.g. `Posts`, `Viewer`, `Events`) should have three core files:

```
/stores
  ├── Posts/
  │   ├── model.ts
  │   ├── schema.ts
  │   └── store.ts
  └── Viewer/
      ├── model.ts
      ├── schema.ts
      └── store.ts
```

---

### 👤 `Viewer` Entity

#### **model.ts**

```ts
import { makeAutoObservable } from 'mobx';
import { type Mapped } from '@stores/core';

export type ViewerDto = { id: number; name: string };

export class ViewerModel {
	constructor(dto: ViewerDto) {
		Object.assign(this, dto);
		makeAutoObservable(this, {}, { autoBind: true });
	}

	get displayName() {
		return this.name.toUpperCase();
	}
}

export interface ViewerModel extends Mapped<ViewerDto> {}
```

#### **schema.ts**

```ts
import { ENTITY_KEY, EntitySchema } from '@stores/core';
import { ViewerModel } from './model';

export const viewerSchema = new EntitySchema(
	ENTITY_KEY.VIEWER,
	{},
	{
		model: ViewerModel,
		idAttribute: props => props?.userId || props?.id,
	}
);
```

---

### 🧱 `Post` Entity

#### **model.ts**

```ts
import { makeAutoObservable } from 'mobx';
import {
	ENTITY_KEY,
	Mapped,
	TEntitiesStore,
	linkEntity,
	getLinkedEntities,
} from '@stores/core';

export class PostDto {
	id!: string;
	userId!: string;
	title!: string;
	body!: string;
	viewer!: { id: string; name: string };
	viewers!: { id: string; name: string }[];
}

export class PostModel {
	constructor(dto: PostDto, entities: TEntitiesStore) {
		linkEntity(this, entities);
		Object.assign(this, dto);
		makeAutoObservable(this, {}, { autoBind: true });
	}

	get author() {
		const entities = getLinkedEntities(this);
		return entities.getEntity(ENTITY_KEY.VIEWER, this.viewerId);
	}

	get shortTitle() {
		return this.title.slice(0, 15) + '…';
	}
}

export interface PostModel extends Mapped<PostDto> {}
```

#### **schema.ts**

```ts
import { ENTITY_KEY, EntitySchema } from '@stores/core';
import { viewerSchema } from '@stores/Viewer/schema';
import { PostModel } from './model';

export const postSchema = new EntitySchema(
	ENTITY_KEY.POST,
	{
		viewer: viewerSchema,
		viewers: [viewerSchema],
	},
	{
		model: PostModel,
	}
);
```

---

### 🧠 Normalized Example

#### 🔸 API response:

```json
{
	"id": "101",
	"title": "Welcome to MobX",
	"body": "Reactive data is magic",
	"viewer": { "id": "10", "name": "Anna" },
	"viewers": [
		{ "id": "11", "name": "John" },
		{ "id": "12", "name": "Maria" },
		{ "id": "13", "name": "Luca" }
	]
}
```

#### 🔸 After normalization:

```ts
entities = {
	posts: {
		'101': {
			id: '101',
			title: 'Welcome to MobX',
			body: 'Reactive data is magic',
			viewerId: '10',
			viewersId: ['11', '12', '13'],
		},
	},
	viewers: {
		'10': { id: '10', name: 'Anna' },
		'11': { id: '11', name: 'John' },
		'12': { id: '12', name: 'Maria' },
		'13': { id: '13', name: 'Luca' },
	},
};
```

---

# 💾 Cache Policy (TTL / Type / Force)

```ts
export const TTL_BY_TYPE = Object.freeze({
	detail: 20_000, // 20s — single card/entity
	list: 60_000, // 60s — list/pagination
	static: 150_000, // 150s — static reference data
	realtime: 2_000, // 2s — frequently updated (status, live feed)
} as const);
```

**Usage:**

```ts
await this.root.api.Posts.getPosts({}, { type: 'list' }); // cached 60s
await this.root.api.Posts.getPosts({}, { type: 'list', force: true }); // force refetch
```

**Rules:**

- No `type` → cache ignored
- `force: true` → bypass cache
- TTL auto-clears expired cache in `ApiManager`

---

# ♻️ Garbage Collector (GC)

The **Garbage Collector** ensures that normalized MobX entities never grow unchecked.
It operates on a **dependency graph** built from `EntitiesStore.snapshot` and applies two cleanup strategies:

- **Graph-based cleanup** → unreachable nodes
- **TTL cleanup** → expired nodes
- **LRU cleanup** → overflow trimming

GC is completely deterministic and fully covered by unit tests.

---

## ⚡️ GC Overview

| Module                    | Responsibility                                             |
| ------------------------- | ---------------------------------------------------------- |
| **GCGraphBuilder**        | Builds dependency graph from normalized entities + schemas |
| **GCGraphWalker**         | Traverses roots/children and detects unreachable nodes     |
| **GCAnalyzer**            | Applies TTL and LRU strategies                             |
| **GarbageCollectorStore** | Public API for running graph/ttl/lru cleanup               |

GC consists of **three phases**:

```
1) Build graph
2) Detect unreachable entities (graph walker)
3) Remove expired or overflow entities (TTL / LRU)
```

---

# 🧱 1. Graph Builder (GCGraphBuilder)

The builder inspects:

- `EntitiesStore.snapshot`
- each entity schema and its relationship fields
- all `xxxId` and `xxxId[]` references
- ensures cross-entity linking (parent ↔ child)

### 🔧 Example Snapshot → Graph

#### Snapshot

```ts
{
  posts: {
    p1: { authorId: 'u1', _meta: { accessedAt: 10 } },
  },
  viewers: {
    u1: { _meta: { accessedAt: 12 } },
  }
}
```

#### Schema

```ts
new EntitySchema('posts', {
	author: viewerSchema,
});
```

#### Graph Output

```ts
{
  posts: {
    p1: {
      parents: [],
      children: [{ key: 'viewers', id: 'u1' }],
      type: 'root',
      meta: { accessedAt: 10, createdAt: 10, updatedAt: 10 }
    }
  },
  viewers: {
    u1: {
      parents: [{ key: 'posts', id: 'p1' }],
      children: [],
      type: 'leaf',
      meta: { accessedAt: 12, createdAt: 12, updatedAt: 12 }
    }
  }
}
```

---

## 🧩 Node Types

| Node Type  | Condition                    | Meaning                          |
| ---------- | ---------------------------- | -------------------------------- |
| `root`     | no parents AND has children  | starting point for reachability  |
| `internal` | has parents AND has children | middle of a chain                |
| `leaf`     | has parents AND no children  | end of a graph chain             |
| `isolated` | no parents AND no children   | unreachable, removed immediately |

---

## 🔄 Builder Pipeline

```
snapshot →
read bucket →
for each entity:
  use schema →
  detect one-to-one / one-to-many →
  resolve xxxId / xxxId[] →
  connect parent/children →
  dedupe links →
  classify node type →
produce GCGraph
```

### Builder Guarantees

- handles cycles (`A → B → A`)
- skips missing id fields
- normalizes numeric/string ids
- deduplicates arrays (`['u1','u1','u2']`)
- missing nodes don't break graph

---

# 🧭 2. Graph Walker (GCGraphWalker)

Walker identifies:

- **reachable nodes**: nodes visited from any root
- **unreachable nodes**: isolated or orphaned nodes

### Output Shape

```ts
{
  roots: { posts: Set([...]), viewers: Set([...]) },
  reachable: { posts: Set([...]), viewers: Set([...]) },
  unreachable: { posts: Set([...]), viewers: Set([...]) }
}
```

Walker is DFS-based and cycle-safe.

---

# ⏳ 3. TTL & LRU Cleanup (GCAnalyzer)

GCAnalyzer applies **two cleanup strategies**:

## 3.1 TTL Strategy

Removes nodes older than their TTL:

```ts
accessedAt < Date.now() - ttl;
```

If missing meta → treated as expired.

TTL is defined in per-entity config:

```ts
POST: { ttl: 1000 },
OTHER: { ttl: 500 },
DEFAULT: { ttl: 1000 }
```

### TTL Output

```ts
{
  POST: Set(["p3", "p8"]),
  VIEWER: Set([])
}
```

---

## 3.2 LRU Strategy

If bucket exceeds `max` entities:

1. Sort by `accessedAt` ASC
2. Remove oldest N entries

Example:

```
max = 2
accessedAt: a=1, b=2, c=3
→ remove "a"
```

Missing meta = timestamp = `0` → removed first.

---

# 🏛️ GarbageCollectorStore — Public API

Main entry for app code:

```ts
processGraph();
processTTL();
processLRU();
runStartup();
```

### `runStartup()`

Runs on app start:

```
1) remove unreachable graph nodes
2) remove expired TTL nodes
```

### `processGraph()`

Removes unreachable nodes.

### `processTTL()`

Removes nodes that expired by TTL.

### `processLRU()`

Trims buckets based on max size cap.

---

# 🧪 Full Test Coverage

The GC system is fully covered with isolated tests:

- TTL tests
- LRU tests
- GraphBuilder deep graph tests
- GraphWalker reachability tests
- Cycles, mixed id types, missing fields
- Boundary timestamp tests
- Degenerate/empty buckets

This ensures deterministic results across all buckets and strategies.

---

# 🧠 Summary

GC provides:

- automatic cleanup
- safe memory footprint
- consistent graph-based reasoning
- deterministic caching behavior
- high performance due to snapshot-based graph building

GC operates entirely on entity snapshots and schemas, independent from MobX internals. MobX only reacts to the resulting entity removals.

# 🔐 Persist Layer

The **Persist Layer** provides durable, versioned, and reactive persistence for:

- plain store state
- normalized entities
- single & multi collections

It is split into four focused parts.

---

## 1️⃣ PersistExtractor — State Reader

Responsible for building a **clean, serializable snapshot** of `RootStore`.

It extracts:

- plain fields from stores (numbers, strings, booleans, POJOs)
- entities snapshot (by entity key)
- collections snapshot (items, hasNoMore, reversed, limit)

It automatically **ignores**:

- functions / methods
- getters / computed values
- collection instances (only their snapshot is stored)
- nested stores / services
- non-serializable objects
- blacklisted stores / fields

---

## 2️⃣ PersistSerializer — JSON ↔ Snapshot

Handles transformation between **snapshot** and **JSON** and restores:

- entities (including `_meta` timestamps)
- single & multi collections
- store plain state

Entities are re-hydrated through their **models**, preserving:

- MobX observability
- meta timestamps (createdAt / updatedAt / accessedAt)
- correct linking via `EntitiesStore.merge()`

---

## 3️⃣ PersistProcessor — Orchestrator & Diff

Coordinates three persistence channels:

- `persist:state`
- `persist:collections`
- `persist:entities`

Responsibilities:

- diff snapshots to avoid redundant writes
- use separate storage keys per channel
- restore in safe order:
  1. state
  2. collections
  3. entities
- wrap restores in MobX actions
- cache last snapshots for equality checks

---

## 4️⃣ PersistStore — Runtime & Versioning

The runtime layer used by the app.

Responsibilities:

- wires extractor + serializer + processor
- manages `isRestoring` flag
- subscribes to:
  - `entities.merge`
  - collection mutations
  - store field changes
- throttles writes to storage
- handles **versioning**:
  - mismatch → full cleanup
  - match → normal restore

```ts
export const PERSIST = {
	KEY: 'persist',
	VERSION: 1,
} as const;
```

Changing `VERSION` invalidates previous snapshots.

---

### 🚀 Bootstrap Restore Flow

On application startup, the Persist system performs a deterministic restore pipeline:

1. **Load snapshot from storage**
2. **Restore plain store state**
   (booleans, numbers, strings, POJOs)
3. **Restore collection snapshots**
   (items, limit, hasNoMore, reversed, pageNumber)
4. **Restore entities + hydrate into models**
5. **Run GC Startup Cleanup**
   - remove unreachable
   - remove expired TTL
6. **Mark restore complete (`isRestoring = false`)**
7. App becomes fully reactive.

This guarantees consistent graph structure, valid model hydration, and clean cache state on startup.

---

# ⚠️ Persist Rules

## 1️⃣ Stores must never mutate state directly

```ts
// ❌ Bad — Persist cannot reliably track this
this.viewer = response;

// ✅ Good — explicit action boundary
this.setViewer(response);
```

---

## 2️⃣ All plain fields must be updated inside actions

```ts
export class ViewerStore {
	private _isLoggedIn = false;
	private _viewer: ViewerDto | null = null;

	constructor(public root: RootStore) {
		makeAutoObservable(this, { root: false });
	}

	setViewer = (viewer: ViewerDto | null) => {
		this._viewer = viewer;
	};

	fetchCurrentViewer = createDuck(async () => {
		const response = await this.root.api.Viewers.getCurrentViewer({
			type: 'static',
		});

		// persist-aware update
		this.setViewer(response);
	});
}
```

---

## 3️⃣ Collections & entities persist automatically

The Persist layer automatically tracks:

- any EntityCollection mutation (`set`, `append`, `prepend`, `reset`, etc.)
- any `EntitiesStore.merge()` call

No manual wiring required.

---

## 4️⃣ Persisted state must remain fully serializable

❌ Forbidden types:

- functions
- classes / instances
- Maps / Sets / Promises
- raw `Date` objects (must be serialized)

✅ Allowed:

- plain DTO-like data
- entity snapshots
- collection snapshots

---

**In short:**
Persist works only when **all meaningful state changes go through explicit actions** that the system can observe.

# 🧩 Usage Guide

## 1️⃣ Multi-list Store (Groups)

### **store.ts**

```ts
export class PostsStore {
	lists: EntityCollection<PostDto, PostModel> &
		Record<string, EntityCollection<PostDto, PostModel>>;

	constructor(public root: RootStore) {
		this.lists = createEntityCollection<PostDto, PostModel>({
			schema: postSchema,
			root,
			limit: 20,
			multi: true,
		});

		makeAutoObservable(this, { root: false });
	}

	fetchPosts = createDuck(
		async ({ group, force }: { group: string; force?: boolean }) => {
			const res = await this.root.api.Posts.getPosts(
				{ _page: 1, _limit: this.lists[group].limit, group },
				{ type: 'list', force }
			);
			this.lists[group].set(res);
		}
	);

	fetchMorePosts = createDuck(async ({ group }: { group: string }) => {
		if (this.lists[group].hasNoMore) return;
		const res = await this.root.api.Posts.getPosts({
			_page: this.lists[group].pageNumber,
			_limit: this.lists[group].limit,
			group,
		});
		this.lists[group].append(res);
	});
}
```

### **React Example (Multi List)**

```tsx
import { useEffect } from 'react';
import { observer } from 'mobx-react-lite';
import { useStore } from '@stores/core';
import Explore from './Explore';

export const GROUPS = { ACTIVE: 'active', PAST: 'past' };

const ExploreContainer = () => {
	const {
		posts: {
			fetchPosts: {
				[GROUPS.ACTIVE]: fetchPostsActive,
				[GROUPS.PAST]: fetchPostsPast,
			},
			fetchMorePosts: {
				[GROUPS.ACTIVE]: fetchMorePostsActive,
				[GROUPS.PAST]: fetchMorePostsPast,
			},
			lists: {
				[GROUPS.ACTIVE]: { getList: activeList },
				[GROUPS.PAST]: { getList: pastList },
			},
		},
	} = useStore();

	const getActive = (force?: boolean) =>
		fetchPostsActive.run({ params: { group: GROUPS.ACTIVE, force } });

	const getPast = (force?: boolean) =>
		fetchPostsPast.run({ params: { group: GROUPS.PAST, force } });

	useEffect(() => {
		getActive();
		getPast();
	}, [getActive, getPast]);

	return (
		<>
			<Explore
				title='Active posts'
				list={activeList}
				isLoading={fetchPostsActive.isLoading}
				isLoadingMore={fetchMorePostsActive.isLoading}
				onRefresh={() => getActive(true)}
				getMorePosts={() =>
					fetchMorePostsActive.run({ params: { group: GROUPS.ACTIVE } })
				}
			/>
			<Explore
				title='Past posts'
				list={pastList}
				isLoading={fetchPostsPast.isLoading}
				isLoadingMore={fetchMorePostsPast.isLoading}
				onRefresh={() => getPast(true)}
				getMorePosts={() =>
					fetchMorePostsPast.run({ params: { group: GROUPS.PAST } })
				}
			/>
		</>
	);
};

export default observer(ExploreContainer);
```

---

## 2️⃣ Single-list Store

```ts
export class PostsStore {
	list: EntityCollection<PostDto, PostModel>;

	constructor(public root: RootStore) {
		this.list = createEntityCollection<PostDto, PostModel>({
			schema: postSchema,
			root,
			limit: 20,
			multi: false,
		});

		makeAutoObservable(this, { root: false });
	}

	fetchPosts = createDuck(async ({ force }?: { force?: boolean }) => {
		const res = await this.root.api.Posts.getPosts(
			{ _page: 1, _limit: this.list.limit },
			{ type: 'list', force }
		);
		this.list.set(res);
	});
}
```

### **React Example (Single List)**

```tsx
import { useEffect } from 'react';
import { observer } from 'mobx-react-lite';
import { useStore } from '@stores/core';
import Explore from './Explore';

const ExploreContainer = () => {
	const {
		posts: { fetchPosts, list },
	} = useStore();

	const getPosts = (force?: boolean) => fetchPosts.run({ params: { force } });

	const onRefresh = () => getPosts(true);

	useEffect(() => {
		getPosts();
	}, [getPosts]);

	return (
		<Explore
			title='Posts'
			list={list.getList}
			isLoading={fetchPosts.isLoading}
			onRefresh={onRefresh}
		/>
	);
};

export default observer(ExploreContainer);
```

---

## ✅ Summary

| Case              | Usage                                                        |
| ----------------- | ------------------------------------------------------------ |
| Multi-list store  | `fetchPosts[group].run(...)` / `fetchPosts[group].isLoading` |
| Single-list store | `fetchPosts.run()` / `fetchPosts.isLoading`                  |
| Refresh           | `force: true`                                                |
| Cached list       | `type: 'list'`                                               |
| Cached detail     | `type: 'detail'`                                             |
| Live updates      | `type: 'realtime'`                                           |

---

✨ **In short:**

- `multi: true` → isolated states per group
- Normalized graph shared across models
- TTL caching integrated
- Full MobX reactivity — zero boilerplate
- Works perfectly with React Native or Web
