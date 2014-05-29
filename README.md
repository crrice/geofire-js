# GeoFire — Realtime location queries with Firebase

GeoFire is a JavaScript library that allows you to store and query a set
of keys based on their geographic location. GeoFire uses Firebase for data
storage, allowing query results to be updated in realtime as they change.
GeoFire does more than just measure the distance between locations; it allows
you to selectively load only the data within certain locations, keeping your
applications light and responsive.

## Downloading

In order to use GeoFire in your project, you need to include the following files in your HTML file:

```html
<!-- RSVP -->
<script src="rsvp.min.js"></script>

<!-- Firebase -->
<script src="firebase.min.js"></script>

<!-- GeoFire -->
<script src="geofire.min.js"></script>
```

You can find each of these files in the `/dest/` directory of this GitHub repository. For debugging purposes, there is also a non-minified `geofire.js` file in the `/dest/` directory.

You can also download all of these files via Bower [__Note__: geofire is currently not available via bower]:

```bash
$ bower install rsvp firebase [geofire]
```

By the time GeoFire version 2.0 is officially released, it will be available via both npm and Bower.

## API Reference

### GeoFire

A `GeoFire` instance is used to read and write geolocation data to your Firebase. You can also use to it create `GeoQuery` objects.

#### new GeoFire(firebaseRef)

Returns a new `GeoFire` instance. The data for this `GeoFire` will be written to the your `firebaseRef` but will not overwrite all of your data at that location. Note that this `firebaseRef` can point to anywhere in your Firebase.

```JavaScript
// Create a Firebase reference where GeoFire will store its information
var dataRef = new Firebase("https://my-firebase.firebaseio-demo.com/");

// Create a GeoFire index
var geoFire = new GeoFire(dataRef);
```

#### GeoFire.set(key, location)

Returns an empty promise fulfilled when the provided `key` - `location` pair has been added to Firebase.

`location` must have the form [latitude, longitude].

`key` must be a string or number.

```JavaScript
geoFire.set("some-unique-key", [37.785326, -122.405696]).then(function() {
  alert("Location has been added to GeoFire");
}, function(error) {
  // Handle error case
});
```

#### GeoFire.get(key)

Returns a promise fulfilled with the `location` corresponding to the provided `key`. If the `key` does not exist, the returned promise is fulfilled with `null`.

The returned location will have the form [latitude, longitude].

`key` must be a string or number.

```JavaScript
geoFire.get("some-unique-key").then(function(location) {
  alert("Provided key has a location of " + location);
}. function(error) {
  // Handle error case
});
```

#### GeoFire.remove(key)

Returns an empty promise fulfilled when the provided `key` has been removed from Firebase. If the the provided `key` is not in this `GeoFire`, the promise will successfully resolve immediately.

This is equivalent to calling `set(key, null)`.

`key` must be a string or number.

```JavaScript
geoFire.remove("some-unique-key").then(function() {
  alert("Location has been removed from GeoFire");
}, function(error) {
  // Handle error case
});
```

#### GeoFire.query(queryCriteria)

Returns a new `GeoQuery` instance with the provide `queryCriteria`.

The `queryCriteria` must be a dictionary containing the following keys:

* `center` - the center of this query with the form [latitude, longitude]
* `radius` - the radius, in kilometers, of this query

```JavaScript
var geoQuery = geoFire.query({
  center: [10.38, 2.41],
  radius: 10.5
});
```

### GeoQuery

A standing query that tracks a set of keys matching a criteria. A new `GeoQuery` is returned every time you call `GeoFire.query()`.

#### GeoQuery.center()

Returns the `location` which marks the center of this query.

The returned `location` will have the form [latitude, longitude].

```JavaScript
var geoQuery = geoFire.query({
  center: [10.38, 2.41],
  radius: 10.5
});

var center = geoQuery.center();  // center === [10.38, 2.41]
```

#### GeoQuery.radius()

Returns the `radius` of this query, in kilometers.

```JavaScript
var geoQuery = geoFire.query({
  center: [10.38, 2.41],
  radius: 10.5
});

var radius = geoQuery.radius();  // radius === 10.5
```

#### GeoQuery.updateCriteria(newQueryCriteria)

Updates the criteria for this query.

`newQueryCriteria` must be a dictionary containing `center`, `radius`, or both.

```JavaScript
var geoQuery = geoFire.query({
  center: [10.38, 2.41],
  radius: 10.5
});

var center = geoQuery.center();  // center === [10.38, 2.41]
var radius = geoQuery.radius();  // radius === 10.5

geoQuery.updateCriteria({
  center: [-50.83, 100.19],
  radius: 5
});

center = geoQuery.center();  // center === [-50.83, 100.19]
radius = geoQuery.radius();  // radius === 5

geoQuery.updateCriteria({
  radius: 7
});

center = geoQuery.center();  // center === [-50.83, 100.19]
radius = geoQuery.radius();  // radius === 7
```

#### GeoQuery.results()

Returns a promise fulfilled with a list of dictionaries containing the `key` - `location` pairs which are currently within this query.

The returned list will have the following form:

```JavaScript
[
  { key: "key1", location: [latitude1, longitude1], distance: distance1 },
  ...
  { key: "keyN", location: [latitudeN, longitudeN], distance: distanceN }
]
```

If there are no keys currently within this query, an empty list will be returned.

```JavaScript
geoQuery.results().then(function(results) {
  results.forEach(function(result) {
    console.log(result.key + " currently in query at " + result.location + " (" + result.distance + " km from center)");
  });
}, function(error) {
  // Handle error case
});
```

#### GeoQuery.on(eventType, callback)

Attaches a `callback` to this query for a given `eventType`. The `callback` will be passed three parameters, the location's key, the location's [latitude, longitude] pair, and this distance from the location to this query's center, in km.

Valid `eventType` values are `key_entered`, `key_left`, and `key_moved`.

`key_entered` is fired when a key enters this query. This can happen when a key moves from a location outside of this query to one inside of it or when a key is written to `GeoFire` for the first time and it falls within this query.

`key_left` is fired when a key moves from a location inside of this query to one outside of it.

`key_moved` is fired when a key which is already in this query moves to another (or the same) location inside of it.

Returns a `GeoCallbackRegistration` which can be used to cancel the `callback`. You can add as many callbacks as you would like by repeatedly calling `on()`. Each one will get called when its corresponding `eventType` fires. Each `callback` must be cancelled individually.

```JavaScript
var onKeyEnteredRegistration = geoQuery.on("key_entered", function(key, location, distance) {
  console.log(key + " entered query at " + location + " (" + distance + " km from center)");
});

var onKeyMovedRegistration = geoQuery.on("key_moved", function(key, location, distance) {
  console.log(key + " moved within query to " + location + " (" + distance + " km from center)");
});

var onKeyLeftRegistration = geoQuery.on("key_left", function(key, location, distance) {
  console.log(key + " left query to " + location + " (" + distance + " km from center)");
});
```

#### GeoQuery.cancel()

Terminates this query so that it no longer sends location updates. All callbacks attached to this query via `on()` will be cancelled. This query can no longer be used in the future.

```JavaScript
var geoQuery = geoFire.query({
  center: [10.38, 2.41],
  radius: 10.5
});

geoQuery.cancel();
```

### GeoCallbackRegistration

An event registration which is used to cancel a `GeoQuery.on()` callback when it is no longer needed. A new `GeoCallbackRegistration` is returned every time you call `GeoQuery.on()`.

#### GeoCallbackRegistration.cancel()

Cancels this `GeoCallbackRegistration` so that it no longer fires its callback.

```JavaScript
// This example stops listening for new keys entering the query once the
// first key leaves the query

var onKeyEnteredRegistration = geoQuery.on("key_entered", function(key, location, distance) {
  console.log(key + " entered query at " + location + " (" + distance + " km from center)");
});

var onKeyLeftRegistration = geoQuery.on("key_left", function(key, location, distance) {
  console.log(key + " left query to " + location + " (" + distance + " km from center)");
  onKeyEnteredRegistration.cancel();
});
```
## Example Usage

```JavaScript
// Create a Firebase reference where GeoFire will store its information
var dataRef = new Firebase("https://my-firebase.firebaseio-demo.com/");

// Create a GeoFire index
var geoFire = new GeoFire(dataRef);

// Add a key to GeoFire
geoFire.set("some-unique-key", [37.78, -122.41]).then(function() {
  // Retrieve the key that was just added to GeoFire
  return geoFire.get("some-unique-key");
}).then(function(location) {
  // location === [37.78, -122.41]
});

// Create a location query for a circle with a 10.5 km radius
var geoQuery = geoFire.query({
  center: [37.60, -121.98],
  radius: 10.5
});

// Get the keys currently in the query
geoQuery.getResults().then(function(results) {
  results.forEach(function(result) {
    console.log(result.key + " currently in query at " + result.location);
  });
});

// Log the results (both initial items and new items that enter into the query)
var onKeyEnteredRegistration = geoQuery.on("key_entered", function(key, location, distance) {
  console.log(key + " entered query at " + location + " (" + distance + " km from center)");
});

// Terminate the query (we will no longer receive location updates from the server for this query)
geoQuery.cancel();
```

## Promises

As can be seen in the example usage above, GeoFire uses promises when writing and retrieving data. Promises represent the result of a potentially long-running operation. Promises do not block execution and act as an object which contains the promised result. Whenever the result has been computed, the promise will call its `then()` method and pass it the result.

GeoFire uses the lightweight [RSVP.js](https://github.com/tildeio/rsvp.js/) library to provide an implementation of JavaScript promises. If you are unfamiliar with promises, please refer to the [RSVP.js documentation](https://github.com/tildeio/rsvp.js/) for all of the details. Here is a quick example of the use of a promise:

```JavaScript
var promise = new RSVP.Promise(function(resolve, reject) {
  var data = getData();
  resolve(data);
});

promise.then(function(result) {
  // Will be called after the promise resolves
  // result will contain the same values as the data passed into resolve() above
}, function(error) {
  // Otherwise, will be called if the promise rejects
})
```

## Contributing

If you'd like to contribute to GeoFire, you'll need to run the following
commands to get your environment set up.

```bash
$ git clone https://github.com/firebase/GeoFire.git
$ npm install    # install local npm build /test dependencies
$ bower install  # install local JavaScript dependencies
$ gulp serve     # watch for file changes and start server
```

`gulp serve` will watch for changes in the `/src/` directory and lint, concatenate, and minify the source files when a change occurs. It also starts a server running GeoFire at `http://localhost:6060`.

During development, you can run the test suite by navigating to `http://localhost:6060/tests/TestRunner.html` or run the tests via the command line using `gulp test`.