# ⚡️ MobX Store Architecture

## 🧠 MobX Store — Description

### 📌 What it is

The **MobX Store** is a reactive and normalized data layer designed to manage network data, caching, and relationships between entities in a clean and predictable way.
It builds on top of **MobX** and provides structure similar to RTK Query or MST — but without boilerplate and with better reactivity.

---

### ⚙️ Core Principles

| Principle                      | Description                                                                            |
| ------------------------------ | -------------------------------------------------------------------------------------- |
| 🧩 **Entity-based data model** | All normalized data is stored in a single `EntitiesStore`, accessible by `ENTITY_KEY`. |
| ⚡️ **Async Ducks**            | Unified async operations with built-in loading/error/data states.                      |
| 🧱 **Entity Schemas**          | Define how objects relate to each other (one-to-one, one-to-many, nested).             |
| 🧠 **Models**                  | Wrap DTOs with computed logic and reactive relationships.                              |
| 📜 **Collections**             | Manage lists of entities with pagination and append/prepend operations.                |
| 🌍 **RootStore**               | Dependency container linking all domain stores (API, Entities, etc.).                  |

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
AsyncDuck.run() → ApiManager.fetch() → normalize(data)
        ↓
EntitiesStore.merge() → EntityCollection updates
        ↓
MobX reactivity → UI updates
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
