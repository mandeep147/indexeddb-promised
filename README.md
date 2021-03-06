# IndexedDB Promised

This is a tiny library that mirrors IndexedDB, but replaces the weird `IDBRequest` objects with promises, plus a couple of other small changes.

## Limitations

### Transaction lifetime

At time of writing, all browsers aside from Chrome don't treat promise callbacks as microtasks, or call microtasks incorrectly. This means transactions end by the time promise callbacks are called. In practice, this means you cannot perform transactions that involve waiting for a value, then using it within the same transaction.

```js
let tx = db.transaction('store', 'readwrite');
let store = db.objectStore('store');
store.get('hello').then(val => store.put(val, 'foo'));
```

The above will fail in browsers other than Chrome, because the transaction has closed by the time we get to the `.put`.

You can work around this in Firefox by using a promise polyfill that correctly uses microtasks, such as [es6-promise](https://github.com/jakearchibald/es6-promise).

### Safari

This is a simple wrapper library, so you're exposed to bugs in the underlying implementation. Unfortunately [Safari has a lot of these](http://www.raymondcamden.com/2014/09/25/IndexedDB-on-iOS-8-Broken-Bad).

## `idb`

This is your entry point to the API. It's exposed to the global scope unless you're using a module system such as browserify, in which case it's the exported object.

### `idb.open(name, version, upgradeCallback)`

This method returns a promise that resolves to a `DB`.

`name` and `version` behave as they do in `indexedDB.open`.

`upgradeCallback` is called if `version` is greater than the version last opened. It's similar to IDB's `onupgradeneeded`. The callback receives an instance of `UpgradeDB`.

```js
idb.open('my-database', 2, upgradeDB => {
  // Note: we don't use 'break' in this switch statement,
  // the fall-through behaviour is what we want.
  switch (upgradeDB.oldVersion) {
    case 0:
      upgradeDB.createObjectStore('key-val');
    case 1:
      upgradeDB.createObjectStore('stuff', {keyPath: ''});
  }
}).then(db => console.log("DB opened!", db));
```

### `idb.delete(name)`

Behaves like `indexedDB.deleteDatabase`, but returns a promise.

```js
idb.delete('my-database').then(_ => console.log('done!'));
```

## `DB`

Properties:

* Same as equivalent properties on an instance of `IDBDatabase`:
  * `name`
  * `version`
  * `objectStoreNames`

Methods:

* `close` - as `idbDatabase.close`
* `transaction` - as `idbDatabase.transaction`, but returns a `Transaction`

## `UpgradeDB`

As `DB`, except:

Properties:

* `transaction` - this is a property rather than a method. It's a `Transaction` representing the upgrade transaction
* `oldVersion` - the previous version of the DB seen by the browser, or 0 if it's new

Methods:

* `createObjectStore` - as `idbDatabase.createObjectStore`, but returns an `ObjectStore`
* `deleteObjectStore` - as `idbDatabase.deleteObjectStore`

## `Transaction`

Properties:

* `complete` - a promise. Resolves when transaction completes, rejects if transaction aborts or errors
* Same as equivalent properties on an instance of `IDBTransaction`:
  * `objectStoreNames`
  * `mode`

Methods:

* `abort` - as `idbTransaction.abort`
* `objectStore` - as `idbTransaction.objectStore`, but returns an `ObjectStore`

```js
idb.open('my-database', 1, upgradeDB => {
  switch (upgradeDB.oldVersion) {
    case 0:
      upgradeDB.createObjectStore('key-val');
  }
}).then(db => {
  let tx = db.transaction('key-val');
  tx.objectStore('key-val').put('hello', 'world');
  return tx.complete;
}).then(_ => console.log("Done!"));
```

## `ObjectStore`

Properties:

* Same as equivalent properties on an instance of `IDBObjectStore`:
  * `name`
  * `keyPath`
  * `indexNames`
  * `autoIncrement`

Methods:

* Same as equivalent methods on an instance of `IDBObjectStore`, but returns a promise that resolves/rejects based on operation success/failure:
  * `put`
  * `add`
  * `delete`
  * `clear`
  * `get`
  * `getAll`
  * `getAllKeys`
  * `count`
* Same as equivalent methods on an instance of `IDBObjectStore`, but returns a promise that resolves with a `Cursor`:
  * `openCursor`
  * `openKeyCursor`
* `deleteIndex` - as `idbObjectStore.deleteIndex`
* Same as equivalent methods on an instance of `IDBObjectStore`, but returns an `Index`: 
  * `createIndex`
  * `index`
* `iterateCursor` - see below
* `iterateKeyCursor` - see below


### `iterateCursor` & `iterateKeyCursor`

Due to the microtask issues in most browsers, iterating over a cursor using promises doesn't really work:

```js
// Currently, this only works in Chrome:
let tx = db.transaction('stuff');
tx.objectStore('stuff').openCursor().then(function cursorIterate(cursor) {
  if (!cursor) return;
  console.log(cursor.value);
  return cursor.continue().then(cursorIterate);
});
tx.complete.then(_ => console.log('done'));
```

So in the mean time, `iterateCursor` and `iterateKeyCursor` map to `openCursor` & `openKeyCursor`, take identical arguments, plus an additional callback that receives an `IDBCursor`, so the above example becomes:

```js
let tx = db.transaction('stuff');
tx.objectStore('stuff').iterateCursor(cursor => {
  if (!cursor) return;
  console.log(cursor.value);
  cursor.continue();
});
tx.complete.then(_ => console.log('done'));
```

The intent is to remove `iterateCursor` and `iterateKeyCursor` from the library once browsers support promises and microtasks correctly.

## `Index`

Properties:

* Same as equivalent properties on an instance of `IDBIndex`:
  * `name`
  * `keyPath`
  * `multiEntry`
  * `unique`

Methods:

* Same as equivalent methods on an instance of `IDBIndex`, but returns a promise that resolves/rejects based on operation success/failure:
  * `get`
  * `getKey`
  * `getAll`
  * `getAllKeys`
  * `count`
* Same as equivalent methods on an instance of `IDBIndex`, but returns a promise that resolves with a `Cursor`:
  * `openCursor`
  * `openKeyCursor`
* `iterateCursor` - as `objectStore.iterateCursor` but over the index
* `iterateKeyCursor` - as `objectStore.iterateKeyCursor` but over the index

## Cursor

Properties:

* Same as equivalent properties on an instance of `IDBCursor`:
  * `direction`
  * `key`
  * `primaryKey`
  * `value`

Methods:

* Same as equivalent methods on an instance of `IDBCursor`, but returns a promise that resolves/rejects based on operation success/failure:
  * `update`
  * `delete`
* Same as equivalent methods on an instance of `IDBCursor`, but returns a promise that resolves with a `Cursor`:
  * `advance`
  * `continue`
  * `continuePrimaryKey`