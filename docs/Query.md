# Query

Queries are the primary way of retrieving data in ApeECS.
Retrieve Entities based on Component composition, specific sets, and reverse references.
Most ECS implementations implement a Union query, which ApeECS does through it's `query.fromAll()` method.

```js
// find all of the entities with both Mass and Point Components/Tags
const entities = world.createQuery().fromAll('Mass', 'Point').execute();
```

👆 A Queries must include at least one from* method call or init option.

👆 `fromAll`, `fromAny`, `fromReverse`, `from`, `not`, `only`, `persist` can all be done in the creation factory's init `Object`.

👆 `from*`, `not`, and `only` methods do not distinguish between Component types and Entity tags.

```js
// functionally equivalent to the previous example
const entities = world.createQuery({ all: ['Mass', 'Point'] }).execute();
```

## create

There are two Query factories -- [world.createQuery](./World.md#createquery) and [system.createQuery](./System.md#createquery).
Each returns a Query instance.
The main difference is that a Query created from a System is associated with that system, and thus can be persisted to track changes.

A common pattern is to create your persisted queries in a System init.

```js
class ApplyMove extends ApeECS.System {

  init() {
    this.moveQuery = this.createQuery().fromAll('Sprite', 'Position', 'MoveAction').persist();
  }

  update(tick) {
    for (const entity of this.moveQuery.execute()) {
      for (const action of entity.getComponents('MoveAction')) {
        entity.c.Position.x += action.x;
        entity.c.Position.y += action.y;
        entity.removeComponent(action);
      }
    }
  }
}
```

### Query init object

You can pass an init object to world or system `createQuery({})`

* init
  * from: `Array`, equiv to from()
  * all: `Array`, equiv to fromAll()
  * any: `Array`, equiv to fromAny()
  * not `Array`, equiv to not()
  * only: `Array`, equiv to only()
  * persist: `bool`, equiv to persist()
  * trackAdd: `bool`, track added entities between system runs on persisted query with the query.added Set.
  * trackRemoved: `bool`, track removed entities between system runs on persisted query with query.removed Set.
  * includeApeDestroy: `bool`, if world.config.useApeDestoy is true, queries default to removing Entities with the `ApeDestroy` tag, but you can include them by setting this to `true`.

## fromAll

Using `fromAll` adds to the Query execution results Entities with at least all of the Component types/Tags listed.
It is literally a set union.

**Arguments**:
* ...types: `[]String|Component class`: _required_, array of strings that are the tags and Component types you require from an entity

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Sprite', 'Position', 'MoveAction');
```

```js
const query = world.createQuery({
  all: ['Sprite', 'Position', 'MoveAction']
});
```

## fromAny

Query `execute` results include Entities with at least one of the tags or Component types listed.

```js
//must have Character Component type or tag and must have one or more of Sprite, Image, or New.
const query = world.createQuery().fromAll('Character').fromAny('Sprite', 'Image', 'New');
```

```js
const query = world.createQuery({
  all: ['Character'],
  any: ['Sprite', 'Image', 'New']
});
```

**Arguments**:
* ...types: `[]String|Component class`: _required_, array of strings that are the tags and Component types you require at least one of from an entity

**Returns**:
* `Query` instance for chaining methods.

## fromReverse

Query `execute` results must include entities that have Components that reference a given entity with a given Component type. This is the inverse of normal queries - instead of finding entities with certain components, it finds entities whose components **reference** a specific target entity.

**Arguments**:
* entity: `Entity|String`, _required_, Entity instance or entity ID that must be referenced by a Component.
* type: `String|Component class`, _required_, Component type name or class that contains the reference to the entity.

**Returns**:
* `Query` instance for chaining methods.

### Basic Usage

```js
// Find all entities that have a component indicating that they're in the player's inventory
const entities = world.createQuery().fromReverse(player, 'InInventory').execute();
```

```js
// Using init object syntax
const query = world.createQuery({
  reverse: {
    entity: player,
    type: 'InInventory'
  }
});
```

### EntityRef Examples

Works with EntityRef properties to find entities that reference a target entity:

```js
class Room extends ApeECS.Component {
  static properties = { name: 'unknown' };
}

class Furniture extends ApeECS.Component {
  static properties = {
    name: 'item',
    room: EntityRef  // Reference to a Room entity
  };
}

world.registerComponent(Room);
world.registerComponent(Furniture);

const livingroom = world.createEntity({
  c: { Room: { name: 'Living Room' }}
});

const sofa = world.createEntity({
  c: { Furniture: { name: 'Sofa', room: livingroom }}
});

const table = world.createEntity({
  c: { Furniture: { name: 'Table', room: livingroom }}
});

// Find all furniture in the living room
const livingroomFurniture = world.createQuery()
  .fromReverse(livingroom, 'Furniture')
  .execute();
// Returns: Set containing sofa and table entities

// Can also use entity ID and component class
const query2 = world.createQuery()
  .fromReverse(livingroom.id, Furniture);
```

### EntitySet Examples

Works with EntitySet properties to find entities whose sets contain a target entity:

```js
class RoomContent extends ApeECS.Component {
  static properties = {
    items: EntitySet  // Set of item entities
  };
}

class Item extends ApeECS.Component {
  static properties = { name: 'item' };
}

const sword = world.createEntity({
  c: { Item: { name: 'Sword' }}
});

const armory = world.createEntity({
  c: { RoomContent: {} }
});

const treasury = world.createEntity({
  c: { RoomContent: {} }
});

// Add sword to armory
armory.c.RoomContent.items.add(sword);

// Find all rooms containing the sword
const roomsWithSword = world.createQuery()
  .fromReverse(sword, 'RoomContent')
  .execute();
// Returns: Set containing armory entity

// Sword can be in multiple rooms
treasury.c.RoomContent.items.add(sword);
const allRoomsWithSword = world.createQuery()
  .fromReverse(sword, 'RoomContent')
  .execute();
// Returns: Set containing both armory and treasury entities
```

### EntityObject Examples

Works with EntityObject properties to find entities whose objects contain a target entity:

```js
class Inventory extends ApeECS.Component {
  static properties = {
    slots: EntityObject  // Object with entity references as values
  };
}

class Item extends ApeECS.Component {
  static properties = { name: 'item' };
}

const sword = world.createEntity({
  c: { Item: { name: 'Magic Sword' }}
});

const warrior = world.createEntity({
  c: { Inventory: {} }
});

const archer = world.createEntity({
  c: { Inventory: {} }
});

// Assign sword to warrior's weapon slot
warrior.c.Inventory.slots['weapon'] = sword;

// Find all inventories containing the sword
const inventoriesWithSword = world.createQuery()
  .fromReverse(sword, 'Inventory')
  .execute();
// Returns: Set containing warrior entity

// Move sword to archer
delete warrior.c.Inventory.slots['weapon'];
archer.c.Inventory.slots['primary'] = sword;

// Same item can be in multiple inventories/slots
warrior.c.Inventory.slots['backup'] = sword;
const allInventoriesWithSword = world.createQuery()
  .fromReverse(sword, 'Inventory')
  .execute();
// Returns: Set containing both warrior and archer entities
```

### Persisted Queries and Updates

**Important**: fromReverse queries automatically update when references change, but behavior differs between persisted and non-persisted queries:

```js
const target = world.createEntity();
const referrer = world.createEntity({
  c: { TestRef: { target: target }}
});

// Persisted query automatically updates after world.tick()
const persistedQuery = world.createQuery()
  .fromReverse(target, 'TestRef')
  .persist();

let results = persistedQuery.execute();
console.log(results.size); // 1

// Change reference
referrer.c.TestRef.target = null;
world.tick(); // Important: call tick to update indexes

results = persistedQuery.execute();
console.log(results.size); // 0 - automatically updated

// Non-persisted query requires manual refresh
const nonPersistedQuery = world.createQuery()
  .fromReverse(target, 'TestRef');

referrer.c.TestRef.target = target; // Re-add reference
world.tick();

results = nonPersistedQuery.execute();
console.log(results.size); // Still 0 - stale results

results = nonPersistedQuery.refresh().execute();
console.log(results.size); // 1 - fresh results after refresh
```

### Performance Considerations

- **Persisted queries**: Automatically updated and indexed. Best for queries used frequently or in systems.
- **Non-persisted queries**: Must be manually refreshed. Best for one-off queries.
- **Reference updates**: All reference changes (EntityRef, EntitySet, EntityObject) trigger index updates for persisted queries.

### Common Pitfalls

⚠️ **Pitfall 1: Non-persisted queries don't auto-update**

```js
const query = world.createQuery().fromReverse(target, 'ComponentType');
// Change references...
world.tick();
const results = query.execute(); // ❌ Stale results!
const freshResults = query.refresh().execute(); // ✅ Fresh results
```

⚠️ **Pitfall 2: Forgetting to call world.tick()**

```js
const query = world.createQuery().fromReverse(target, 'ComponentType').persist();
entity.c.ComponentType.ref = newTarget;
const results = query.execute(); // ❌ Still shows old results
world.tick(); // ✅ Must call tick to update indexes
const updatedResults = query.execute(); // ✅ Now shows correct results
```

⚠️ **Pitfall 3: Entity destruction edge cases**

```js
// Target entity destroyed - query still works but returns empty results
target.destroy();
const results = query.execute(); // Returns empty Set (not error)

// Referrer entity destroyed - automatically removed from query results
referrer.destroy();
world.tick();
const results2 = query.execute(); // Automatically excludes destroyed entity
```

## from

Limit the Query `execute` results to only include a subset of these specified entities.

**Arguments**:
* ...entities: `[]Entity`, _required_, Array of entity lists that is a superset of the results

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().from([player, enemy1]);
```

```js
const query = world.createQuery({
  from: [player, enemy1]
});
```

## not

Limit Query `execute` results to not include Entities that have any of these Component types or tags.
`not()` filters results, and thus the query must include at least one of `from`, `fromAll` or `fromAny` as well.

**Arguments**:
* ...types: `[]String|Component class`, _required_, Array of Component types and Tags to disqualify result entities

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Character', 'Sprite').not('Invisible', 'MarkedForRemoval');
```

```js
const query = world.createQuery({
  all: ['Character', Sprite],
  not: ['Invisible', 'MarkedForRemoval']
});
```

## only

Limit Query `execute` results to only include Entities that have at least one of these Component types or tags.
`only()` filters results, and thus the query must include at least one of `from`, `fromAll` or `fromAny` as well.

**Arguments**:
* ...types: `[]String|Component class`, _required_, Array of Component types and Tags to disqualify result entities

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Character', 'Sprite').only('Invisible', 'MarkedForRemoval');
```

```js
const query = world.createQuery({
  all: ['Character', Sprite],
  only: ['Invisible', 'MarkedForRemoval']
});
```

## persist

Indicate that the Query should be persisted, turning it into a live index of Entities.

**Arguments**:
* trackAdded: `Boolean`, _optional_, flag to track new Entity results from the Query since the last `system.update`
* trackRemoved: `Boolean`, _optional_, flag to track removed Entity results from the Query since the last `system.update`

The properties `query.added` and `query.removed` are `Sets` that you can check during your `system.update` if tracked.

👆 A query can be persisted without having to track added or removed.
Whenever an Entity changes Component or Tag composition, it's checked against all persisted Queries when the `world.tick()` or after a `system.update(tick)` happens.

👆 Peristed queries only update their results after `system.update` or during `world.tick()`.
If you want to update your persisted queries at other times, run [world.updateIndexes()](./World.md#updateindexes).


⚠ Queries cannot be persisted if they use `from` a static set of Entities, or if they're not created from a System.

💭 If you persist a LOT of Queries, it can have a performance from creating Entities, or adding/removing Components or Tags.

## refresh

[execute](#execute) will not update results for changed entities, and persisted queries won't update within a single system update. To get new results in these situations, use refresh.

```js
const world.registerTags('A', 'B');
const entity1 = world.createEntity({ tags: ['A'] });
const query = world.createQuery().fromAll('A', 'B');
const results1 = query.execute(); // doesn't include entity1
entity1.addTag('B');
const results2 = query.execute(); // doesn't include entity1
const result3 = query.refresh().execute(); //does include entity1
```

👆 After each system run and each tick, [world.updateIndexes()](./World.md#updateindexes) which will refresh persisted queries from changed entities.
You can run this directly as well.

## execute

Execute a Query, returning all of the resulting Entities.

**Arguments**:
* filter: `Object`, _optional_, Filter Entities to results that had Component/tag composition changes since `updatedComponents` or Component value changes since `updatedValues`.

```js
const query = world.createQuery().fromAll('Character', 'Sprite');
//only include entities that have been updated last tick or this tick
const entities = query.execute({
  updatedComponents: world.currentTick - 1,
  updatedValues: world.currentTick - 1
})
```

⚠ If you neglect to call [component.update()](./Component.md#update) when you update the values of a Component, then `component.updated` and `entity.updatedValues` will not be updated, and the query filter for `updatedValues` will not be accurate.
