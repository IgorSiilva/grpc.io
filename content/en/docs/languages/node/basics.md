---
title: Basics tutorial
description: A basic tutorial introduction to gRPC in Node.
weight: 50
spelling: cSpell:ignore Protobuf oneofs COORD
---

This tutorial provides a basic Node.js programmer's introduction
to working with gRPC.

By walking through this example you'll learn how to:

- Define a service in a `.proto` file.
- Use the Node.js gRPC API to write a simple client and server for your service.

It assumes that you have read the [Introduction to gRPC](/docs/what-is-grpc/introduction/) and are familiar
with [protocol
buffers](https://protobuf.dev/overview). Note
that the example in this tutorial uses the
[proto3](https://github.com/google/protobuf/releases) version of the protocol
buffers language. You can find out more in the
[proto3 language guide](https://protobuf.dev/programming-guides/proto3).

### Why use gRPC?

{{< why-grpc >}}

### Example code and setup

The example code for our tutorial is in
[grpc/grpc-node/examples/routeguide/dynamic_codegen](https://github.com/grpc/grpc-node/tree/{{< param grpc_vers.node >}}/examples/routeguide/dynamic_codegen).
As you'll see if you look at the repository, there's also a very similar-looking
example in
[grpc/grpc-node/examples/routeguide/static_codegen](https://github.com/grpc/grpc-node/tree/{{< param grpc_vers.node >}}/examples/routeguide/static_codegen).
We have two versions of our route guide example because there are two ways to
generate the code needed to work with protocol buffers in Node.js - one approach
uses `Protobuf.js` to dynamically generate the code at runtime, the other uses
code statically generated using the protocol buffer compiler `protoc`. The
examples behave identically, and either server can be used with either client.
As suggested by the directory name, we'll be using the version with dynamically
generated code in this document, but feel free to look at the static code
example too.

To download the example, clone the `grpc` repository by running the following
command:

```sh
$ git clone -b {{< param grpc_vers.node >}} --depth 1 --shallow-submodules https://github.com/grpc/grpc-node
$ cd grpc
```

Then change your current directory to `examples`:

```sh
$ cd examples
```

You also should have the relevant tools installed to generate the server and
client interface code - if you don't already, follow the setup instructions in
[Quick start](../quickstart/).


### Defining the service

Our first step (as you'll know from the [Introduction to gRPC](/docs/what-is-grpc/introduction/)) is to
define the gRPC *service* and the method *request* and *response* types using
[protocol
buffers](https://protobuf.dev/overview). You can
see the complete .proto file in
[`examples/protos/route_guide.proto`](https://github.com/grpc-node/grpc/blob/{{< param grpc_vers.node >}}/examples/protos/route_guide.proto).

To define a service, you specify a named `service` in your `.proto` file:

```protobuf
service RouteGuide {
   ...
}
```

Then you define `rpc` methods inside your service definition, specifying their
request and response types. gRPC lets you define four kinds of service methods,
all of which are used in the `RouteGuide` service:

- A *simple RPC* where the client sends a request to the server using the stub
  and waits for a response to come back, just like a normal function call.

  ```protobuf
  // Obtains the feature at a given position.
  rpc GetFeature(Point) returns (Feature) {}
  ```

- A *server-side streaming RPC* where the client sends a request to the server
  and gets a stream to read a sequence of messages back. The client reads from
  the returned stream until there are no more messages. As you can see in our
  example, you specify a server-side streaming method by placing the `stream`
  keyword before the *response* type.

  ```protobuf
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  ```

- A *client-side streaming RPC* where the client writes a sequence of messages
  and sends them to the server, again using a provided stream. Once the client
  has finished writing the messages, it waits for the server to read them all
  and return its response. You specify a client-side streaming method by placing
  the `stream` keyword before the *request* type.

  ```protobuf
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  ```

- A *bidirectional streaming RPC* where both sides send a sequence of messages
  using a read-write stream. The two streams operate independently, so clients
  and servers can read and write in whatever order they like: for example, the
  server could wait to receive all the client messages before writing its
  responses, or it could alternately read a message then write a message, or
  some other combination of reads and writes. The order of messages in each
  stream is preserved. You specify this type of method by placing the `stream`
  keyword before both the request and the response.

  ```protobuf
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
  ```

Our `.proto` file also contains protocol buffer message type definitions for all
the request and response types used in our service methods - for example, here's
the `Point` message type:

```protobuf
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

### Loading service descriptors from proto files

The Node.js library dynamically generates service descriptors and client stub
definitions from `.proto` files loaded at runtime.

To load a `.proto` file, simply `require` the gRPC proto loader library and use its
`loadSync()` method, then pass the output to the gRPC library's `loadPackageDefinition` method:

```js
var PROTO_PATH = __dirname + '/../../protos/route_guide.proto';
var grpc = require('@grpc/grpc-js');
var protoLoader = require('@grpc/proto-loader');
// Suggested options for similarity to existing grpc.load behavior
var packageDefinition = protoLoader.loadSync(
    PROTO_PATH,
    {keepCase: true,
     longs: String,
     enums: String,
     defaults: true,
     oneofs: true
    });
var protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
// The protoDescriptor object has the full package hierarchy
var routeguide = protoDescriptor.routeguide;
```

Once you've done this, the stub constructor is in the `routeguide` namespace
(`protoDescriptor.routeguide.RouteGuide`) and the service descriptor (which is
used to create a server) is a property of the stub
(`protoDescriptor.routeguide.RouteGuide.service`);

### Creating the server {#server}

First let's look at how we create a `RouteGuide` server. If you're only
interested in creating gRPC clients, you can skip this section and go straight
to [Creating the client](#client) (though you might find it interesting
anyway!).

There are two parts to making our `RouteGuide` service do its job:
- Implementing the service interface generated from our service definition:
  doing the actual "work" of our service.
- Running a gRPC server to listen for requests from clients and return the
  service responses.

You can find our example `RouteGuide` server in
[examples/routeguide/dynamic_codegen/route_guide_server.js](https://github.com/grpc/grpc-node/blob/{{< param grpc_vers.node >}}/examples/routeguide/dynamic_codegen/route_guide_server.js).
Let's take a closer look at how it works.

#### Implementing RouteGuide

As you can see, our server has a `Server` constructor generated from the
`RouteGuide.service` descriptor object

```js
var Server = new grpc.Server();
```
In this case we're implementing the *asynchronous* version of `RouteGuide`,
which provides our default gRPC server behavior.

The functions in `route_guide_server.js` implement all our service methods.
Let's look at the simplest type first, `getFeature`, which just gets a `Point`
from the client and returns the corresponding feature information from its
database in a `Feature`.

```js
function checkFeature(point) {
  var feature;
  // Check if there is already a feature object for the given point
  for (var i = 0; i < feature_list.length; i++) {
    feature = feature_list[i];
    if (feature.location.latitude === point.latitude &&
        feature.location.longitude === point.longitude) {
      return feature;
    }
  }
  var name = '';
  feature = {
    name: name,
    location: point
  };
  return feature;
}
function getFeature(call, callback) {
  callback(null, checkFeature(call.request));
}
```

The method is passed a call object for the RPC, which has the `Point` parameter
as a property, and a callback to which we can pass our returned `Feature`. In
the method body we populate a `Feature` corresponding to the given point and
pass it to the callback, with a null first parameter to indicate that there is
no error.

Now let's look at something a bit more complicated - a streaming RPC.
`listFeatures` is a server-side streaming RPC, so we need to send back multiple
`Feature`s to our client.

```js
function listFeatures(call) {
  var lo = call.request.lo;
  var hi = call.request.hi;
  var left = _.min([lo.longitude, hi.longitude]);
  var right = _.max([lo.longitude, hi.longitude]);
  var top = _.max([lo.latitude, hi.latitude]);
  var bottom = _.min([lo.latitude, hi.latitude]);
  // For each feature, check if it is in the given bounding box
  _.each(feature_list, function(feature) {
    if (feature.name === '') {
      return;
    }
    if (feature.location.longitude >= left &&
        feature.location.longitude <= right &&
        feature.location.latitude >= bottom &&
        feature.location.latitude <= top) {
      call.write(feature);
    }
  });
  call.end();
}
```

As you can see, instead of getting the call object and callback in our method
parameters, this time we get a `call` object that implements the `Writable`
interface. In the method, we create as many `Feature` objects as we need to
return, writing them to the `call` using its `write()` method. Finally, we call
`call.end()` to indicate that we have sent all messages.

If you look at the client-side streaming method `RecordRoute` you'll see it's
quite similar to the unary call, except this time the `call` parameter
implements the `Reader` interface. The `call`'s `'data'` event fires every time
there is new data, and the `'end'` event fires when all data has been read. Like
the unary case, we respond by calling the callback

```js
call.on('data', function(point) {
  // Process user data
});
call.on('end', function() {
  callback(null, result);
});
```

Finally, let's look at our bidirectional streaming RPC `RouteChat()`.

```js
function routeChat(call) {
  call.on('data', function(note) {
    var key = pointKey(note.location);
    /* For each note sent, respond with all previous notes that correspond to
     * the same point */
    if (route_notes.hasOwnProperty(key)) {
      _.each(route_notes[key], function(note) {
        call.write(note);
      });
    } else {
      route_notes[key] = [];
    }
    // Then add the new note to the list
    route_notes[key].push(JSON.parse(JSON.stringify(note)));
  });
  call.on('end', function() {
    call.end();
  });
}
```

This time we get a `call` implementing `Duplex` that can be used to read *and*
write messages. The syntax for reading and writing here is exactly the same as
for our client-streaming and server-streaming methods. Although each side will
always get the other's messages in the order they were written, both the client
and server can read and write in any order — the streams operate completely
independently.

#### Starting the server

Once we've implemented all our methods, we also need to start up a gRPC server
so that clients can actually use our service. The following snippet shows how we
do this for our `RouteGuide` service:

```js
function getServer() {
  var server = new grpc.Server();
  server.addService(routeguide.RouteGuide.service, {
    getFeature: getFeature,
    listFeatures: listFeatures,
    recordRoute: recordRoute,
    routeChat: routeChat
  });
  return server;
}
var routeServer = getServer();
routeServer.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  routeServer.start();
});
```

As you can see, we build and start our server with the following steps:

 1. Create a `Server` constructor from the `RouteGuide` service descriptor.
 1. Implement the service methods.
 1. Create an instance of the server by calling the `Server` constructor with
    the method implementations.
 1. Specify the address and port we want to use to listen for client requests
    using the instance's `bind()` method.
 1. Call `start()` on the instance to start the RPC server.

### Creating the client {#client}

In this section, we'll look at creating a Node.js client for our `RouteGuide`
service. You can see our complete example client code in
[examples/routeguide/dynamic_codegen/route_guide_client.js](https://github.com/grpc/grpc-node/blob/{{< param grpc_vers.node >}}/examples/routeguide/dynamic_codegen/route_guide_client.js).

#### Creating a stub

To call service methods, we first need to create a *stub*. To do this, we just
need to call the RouteGuide stub constructor, specifying the server address and
port.

```js
new routeguide.RouteGuide('localhost:50051', grpc.credentials.createInsecure());
```

#### Calling service methods

Now let's look at how we call our service methods. Note that all of these
methods are asynchronous: they use either events or callbacks to retrieve
results.

##### Simple RPC

Calling the simple RPC `GetFeature` is nearly as straightforward as calling a
local asynchronous method.

```js
var point = {latitude: 409146138, longitude: -746188906};
stub.getFeature(point, function(err, feature) {
  if (err) {
    // process error
  } else {
    // process feature
  }
});
```

As you can see, we create and populate a request object. Finally, we call the
method on the stub, passing it the request and callback. If there is no error,
then we can read the response information from the server from our response
object.

```js
console.log('Found feature called "' + feature.name + '" at ' +
    feature.location.latitude/COORD_FACTOR + ', ' +
    feature.location.longitude/COORD_FACTOR);
```

##### Streaming RPCs

Now let's look at our streaming methods. If you've already read [Creating the
server](#server) some of this may look very familiar - streaming RPCs are
implemented in a similar way on both sides. Here's where we call the server-side
streaming method `ListFeatures`, which returns a stream of geographical
`Feature`s:

```js
var call = client.listFeatures(rectangle);
  call.on('data', function(feature) {
      console.log('Found feature called "' + feature.name + '" at ' +
          feature.location.latitude/COORD_FACTOR + ', ' +
          feature.location.longitude/COORD_FACTOR);
  });
  call.on('end', function() {
    // The server has finished sending
  });
  call.on('error', function(e) {
    // An error has occurred and the stream has been closed.
  });
  call.on('status', function(status) {
    // process status
  });
```

Instead of passing the method a request and callback, we pass it a request and
get a `Readable` stream object back. The client can use the `Readable`'s
`'data'` event to read the server's responses. This event fires with each
`Feature` message object until there are no more messages. Errors in the `'data'`
callback will not cause the stream to be closed. The `'error'` event
indicates that an error has occurred and the stream has been closed. The
`'end'` event indicates that the server has finished sending and no errors
occurred. Only one of `'error'` or `'end'` will be emitted. Finally, the
`'status'` event fires when the server sends the status.

The client-side streaming method `RecordRoute` is similar, except there we pass
the method a callback and get back a `Writable`.

```js
var call = client.recordRoute(function(error, stats) {
  if (error) {
    callback(error);
  }
  console.log('Finished trip with', stats.point_count, 'points');
  console.log('Passed', stats.feature_count, 'features');
  console.log('Travelled', stats.distance, 'meters');
  console.log('It took', stats.elapsed_time, 'seconds');
});
function pointSender(lat, lng) {
  return function(callback) {
    console.log('Visiting point ' + lat/COORD_FACTOR + ', ' +
        lng/COORD_FACTOR);
    call.write({
      latitude: lat,
      longitude: lng
    });
    _.delay(callback, _.random(500, 1500));
  };
}
var point_senders = [];
for (var i = 0; i < num_points; i++) {
  var rand_point = feature_list[_.random(0, feature_list.length - 1)];
  point_senders[i] = pointSender(rand_point.location.latitude,
                                 rand_point.location.longitude);
}
async.series(point_senders, function() {
  call.end();
});
```

Once we've finished writing our client's requests to the stream using `write()`,
we need to call `end()` on the stream to let gRPC know that we've finished
writing. If the status is `OK`, the `stats` object will be populated with the
server's response.

Finally, let's look at our bidirectional streaming RPC `routeChat()`. In this
case, we just pass a context to the method and get back a `Duplex` stream
object, which we can use to both write and read messages.

```js
var call = client.routeChat();
```

The syntax for reading and writing here is exactly the same as for our
client-streaming and server-streaming methods. Although each side will always
get the other's messages in the order they were written, both the client and
server can read and write in any order — the streams operate completely
independently.

### Try it out!

Build the client and server:

```sh
$ npm install
```

Run the server:

```sh
$ node ./routeguide/dynamic_codegen/route_guide_server.js --db_path=./routeguide/dynamic_codegen/route_guide_db.json
```

From a different terminal, run the client:

```sh
$ node ./routeguide/dynamic_codegen/route_guide_client.js --db_path=./routeguide/dynamic_codegen/route_guide_db.json
```
