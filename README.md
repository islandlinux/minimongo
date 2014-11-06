# Minimongo

A client-side MongoDB implementation which supports basic queries, including some geospatial ones.

Uses code from Meteor.js minimongo package, reworked to support more geospatial queries and made npm+browserify friendly. It was forked in January 2014.

It is either IndexedDb backed (IndexedDb), WebSQL backed (WebSQLDb), Local storage backed (LocalStorageDb) or in memory only (MemoryDb).

Autoselection is possible with utils.autoselectLocalDb(options, success, error)

## Usage

Minimongo is designed to be used with browserify.

```javascript

// Require minimongo
var minimongo = require("minimongo");

var LocalDb = minimongo.MemoryDb;

// Create local db (in memory database with no backing)
db = new LocalDb();

// Add a collection to the database
db.addCollection("animals");

doc = { species: "dog", name: "Bingo" };

// Always use upsert for both inserts and modifies
db.animals.upsert(doc, function() {
	// Success:

	// Query dog (with no query options beyond a selector)
	db.animals.findOne({ species:"dog" }, {}, function(res) {
		console.log("Dog's name is: " + res.name);
	});
});
```

### Upserting

`db.sometable.upsert(doc, success, error)` can take either a single document or multiple documents (array) for the first parameter.

*Note*: Multiple-upsert only applies to local databases for now, not RemoteDb

### Resolving upserts

Upserts are stored in local databases in a special state to record that they are upserts, not cached rows. The base document on which the upsert is based is also stored. For example, if a row starts in cached state with `{ x:1 }` and is upserted to `{ x: 2 }`, both the upserted and the original state are stored. This allows the server to do 3-way merging and apply only the changes.

To resolve the upsert (for example once sent to central db), use resolveUpsert on collection

`db.sometable.resolveUpsert(doc, success, error)` can take either a single document or multiple documents (array) for the first parameter.

`resolveUpsert` takes the `doc` parameter which is the document that was the local upserted row, not the merged value from the server. It is used to determine if another upsert has taken place since. If another upsert has taken place, the base value is updated (since the change has been accepted by the server) but the new upserted value is left alone. 

### IndexedDb

To make a database backed by IndexedDb:

```javascript

// Require minimongo
var minimongo = require("minimongo");

var IndexedDb = minimongo.IndexedDb;

// Create IndexedDb
db = new IndexedDb({namespace: "mydb"}, function() {
	// Add a collection to the database
	db.addCollection("animals", function() {
		doc = { species: "dog", name: "Bingo" };

		// Always use upsert for both inserts and modifies
		db.animals.upsert(doc, function() {
			// Success:

			// Query dog (with no query options beyond a selector)
			db.animals.findOne({ species:"dog" }, {}, function(res) {
				console.log("Dog's name is: " + res.name);
			});
		});
	});
}, function() { alert("some error!"); });

```

### Caching

Rows can be cached without creating a pending upsert. This is done automatically when HybridDb uploads to a remote database with the returned upserted rows. It is also done when a query is performed on HybridDb: the results are cached in the local db and the query is re-performed on the local database.

The field `_rev`, if present is used to prevent overwriting with older versions. This is the odd scenario where an updated version of a row is present, but an older query to the server is delayed in returning. To prevent this race condition from giving stale data, the _rev field is used.

### HybridDb

Queries the local database first and then returns remote data if different than local version. 

This approach allows fast responses but with subsequent correction if the server has differing information.

The HybridDb collections can also be created in non-caching mode, which is useful for storing up changes to be sent to a sever:

```
db.addCollection("sometable", { caching: false })
```

To keep a local database and a remote database in sync, create a HybridDb:

```
hybridDb = new HybridDb(localDb, remoteDb)
```

Be sure to add the same collections to all three databases (local, hybrid and remote).

Then query the hybridDb (`find` and `findOne`) to have it get results and correctly combine them with any pending local results. If you are not interested in caching results, add `{ mode: "remote" }` to the options of `find` or `findOne`

When upserts and removes are done on the HybridDb, they are queued up in the LocalDb until `hybridDb.upload(success, error)` is called.

`upload` will go through each collection and send any upserts or removes to the remoteDb.

### RemoteDb

Uses AJAX-JSON calls to an API to query a real Mongo database. API is simple and contains only query, upsert and remove commands.

