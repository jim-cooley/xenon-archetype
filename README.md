
# Overview
Xenon is a framework for writing small REST-based services. (Some people call them microservices.) The runtime is implemented in Java and acts as the host for the lightweight, asynchronous services. The programming model is language agnostic (does not rely on Java specific constructs) so implementations in other languages are encouraged. The services can run on a set of distributed nodes. Xenon provides replication, synchronization, ordering, and consistency for the state of the services. Because of the distributed nature of Xenon, the services scale well and are highly available.

Xenon is a "batteries included" framework. Unlike some frameworks that provide just consistent data replication or just a microservice framework, Xenon provides both. Xenon services have REST-based APIs and are backed by a consistent, replicated document store.

Each service has less than 500 bytes of overhead and can be paused/resumed, making Xenon able to host millions of service instances even on a memory constrained environment.

Service authors annotate their services with various service options, acting as requirements on the runtime, and the framework implements the appropriate algorithms to enforce them. The runtime exposes each service with a URI and provides utility services per instance, for stats, reflection, subscriptions and configuration. A built-in load balancer routes client requests between nodes, according to service options and plug-able node selection algorithms. Xenon supports multiple, independent node groups, maintaining node group state using a scalable gossip scheme.

A powerful index service, invoked as part of the I/O pipeline for persisted services, provides a multi version document store with a rich query language.

High availability and scale-out is enabled through the use of a consensus and replication algorithm and is also integrated in the I/O processing.

# What can Xenon be used for?
The lightweight runtime enables the creation of highly available and scalable applications in the form of cooperating light weight services. The operation model for a cluster of Xenon nodes is the same for both on premise, and service deployments.

The photon controller project makes heavy use of Xenon to build a scalable and highly available Infrastructure-as-a-Service fabric, composed of stateful services, responsible for configuration (desired state), work flows (finite state machine tasks), grooming and scheduling logic. Xenon is also used by several teams building new products, services and features, within VMware.

# Learning More
For a hands-on walkthrough of the sample code contained in this project, keep reading!

For more technical details including tutorials, please refer to the wiki.

# Reporting Issues
Xenon uses a public pivotal tracker project for tracking and reporting issues: https://www.pivotaltracker.com/n/projects/1471320

# Quickstart
This is the sample project for a Xenon Host and associated services.  Here, we'll walk through getting the sample up and running and then provide some background reading for you to increase your understanding of what Xenon is and how it works.

The service contained here is the sample service from the Xenon repository.  We'll walk you through getting that project up and then help you customize and build your first Xenon application.  Later, this guide will provide some useful pointers to additional information and key information that will help you manage and debug your Xenon application.    

## 
There are multiple definitions of "services", so let's define what we mean by a Xenon service. (For now, we’ll assume we’re talking about Xenon running on a single host--we'll get to using multiple hosts in a moment.)

A single Xenon service implements a REST-based API for a single URI endpoint. For example, you might have a service running on:

https://myhost.example.com/example/service
You have a lot of choices in how you implement the service, but a few things are true:

The service can support any of the standard REST operations: GET, POST, PUT, PATCH, DELETE

This service has a document associated with it. If you do a GET on the service, you’ll see the document. In most cases, the document is represented in JSON. There are a few standard JSON fields that every document has, such as the the "kind" of document, the version, the time it was last updated, etc.

Your service may have business logic associated with it. Perhaps when you do a POST, it creates a new VM, and a PATCH will modify the VM.

A service may have its state persisted on disk (a "stateful" service) or it may be generated on the fly (a “stateless” service). Persisted services can have an optional expiration time, after which they are removed from the datastore.

All services can communicate with all other services by using the same APIs that a client will use. Within a single host, communication is optimized (no need to use the network). API calls from clients or services are nearly identical and treated the same way. This makes for a very consistent communication pattern.

For stateful services, only one modification may happen at a time: modifications to a statefule service are serialized, while reads from a service can be made in parallel. If two modifications are attempted in parallel, there will be a conflict. One will succeed, and the other will receive an error. (Xenon is flexible, and allows you to disable this if you really want to.)

## Building the Sample

Build the sample application with the following commands issued from your project root directory:
```sh
$ mvn clean
```

## Starting the Xenon host

The sample host will listen on port 8000 by default; Let's start the sample host:
```sh
$ java -jar xenon-host/target/xenon-host-*-with-dependencies.jar
```

## Sample Code Walkthrough
Now lets walk through the sample code and see what is happening underneath the hood.

### The service factory
A client can create new instances of a service through another service, the service factory. A service factory mostly relies on the Xenon framework for doing all the work and has the following properties:
 * it is stateless - A factory relies 100% on the document index or service host for keeping track of any child service instances. The lifecycle of a child is controlled by DELETE requests directly to the child.
 * GET requests are queries - The factory simply forwards a GET to its URI, to the document index, which returns all documents that their selfLink is prefixed by the factory URI

### Creating a factory

A developer can either derive from the FactoryService abstract class, or use a static helper to create a default instance:
 Below is a minimal, but still useful, factory:
```java
    /**
     * Create a default factory service that starts instances of this service on POST.
     * This method is optional, {@code FactoryService.create} can be used directly
     */
    public static FactoryService createFactory() {
        return FactoryService.createIdempotent(ExampleService.class);
    }
```

Since factory code gives limited ability to change POST or GET processing, deriving from the factory class is not recommended. In particular, **avoid any state side effects across services** in replicated factory handlePost methods. Most of the logic that deals with service creation happens on POST completion, and on the owner node, so you will be violating key invariants.

### Starting a Factory service

The factory service is a singleton that should be started on host start:
```java
    @Override
    public ServiceHost start() throws Throwable {
        super.start();

        startDefaultCoreServicesSynchronously();

        setAuthorizationContext(this.getSystemAuthorizationContext());

        // Start the example service factory
        super.startFactory(ExampleService.class, ExampleService::createFactory);

       ...
       ...
```


### Per service Utility URIs
Each service comes with a set of Xenon provided [utility services](./REST-API#helper-services), listening under the service/* suffix. Use them to gather stats, subscribe, or interact with your service through a browser

## POST Handling
The factory service does not need to implement any handlers, if no additional logic or validation is required for processing POST requests. The _FactoryService_ super class takes care of POST handling, and also GET. The child service URI will be the composite of the factory URI (the prefix) and whatever selfLink path was supplied in the POST body. If no body was supplied, a random UUID will be used for the child.

### Durability
If the service instance state must survive host restarts, the child service instance must enable the ***ServiceOption.PERSISTENCE*** in its constructor. The runtime will then make sure every state change, including service creation is indexed on disk, and on restart, it will automatically re-create all child services, per factory and set the version numbers to the latest known version per child service instance

Assuming the service host is running locally (see the developer guide and debugging page), first do a GET on the example factory, to verify no example service instances are created:
```sh
$ curl http://localhost:8000/core/examples
```

Alternatively you can use the Xenon cli [xenonc](./Xenon-Client-(xenonc))

```sh
$ export XENON=http://localhost:8000
$ xenonc get /core/examples
```

The factory responds with:

```json
{
  "documentLinks": [],
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "168c9af7-193d-4b6c-92c5-409df5b27752"
}
```

The response above means no documents / services are created yet.

### POST Example

Create a new instance by issuing a POST. If no body is supplied the child service will have a default initial state and a random selflink.

```sh
$ curl -X POST -H "Content-type: application/json" -d '{"documentSelfLink":"niki","name":"n"}' http://localhost:8000/core/examples
```

Same operation, using xenonc
```sh
$ xenonc post /core/examples --documentSelfLink=niki --name=n
```

The example factory will respond with the initial state of the new service:

```json
{
  "keyValues": {},
  "name": "n",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/niki",
  "documentSignature": "fdaccd2541efc842e42fb9693be00dbc731dea07",
  "documentUpdateTimeMicros": 1432745225186007,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "31716c5e-393b-4ca7-a15c-af02af0d6d5e"
}
```

Doing another GET on the factory URI shows the new child service

```sh
$ curl http://localhost:8000/core/examples
{
  "documentLinks": [
    "/core/examples/niki"
  ],
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "31716c5e-393b-4ca7-a15c-af02af0d6d5e"
}
```

### GET Handling
The super class handles GET requests to the factory URI and converts them to a document index query. The client can add the _expand_ URI parameter to request embedding the child service state as part of the result

### The child or singleton service

A service is called a singleton if its has a fixed URI and only one instance can exist per host. Otherwise, it just a child service, where many instances can co-exist under some shared URI prefix.

### PATCH handler
The PATCH handler above will use the body associated with the request (patch.getBody()) and the latest state of the service, associated with the patch (getState(patch)). Using fields in the patch body, it will update the current state and the complete the operation.

The service handlers are 100% asynchronous. This means that the handler method can exit, and the client **will not see the operation complete**. An operation is only completed when the service code calls
```java
operation.complete();
```
The service handler method can issue an asynchronous request to another service, return from the method, and only one the secondary request completes, complete its operation.
### CURL PATCH Example

Using curl once again, create a minimal JSON body with a name property and send it to the child service. Use the URI returned from the POST when creating the service, in the documentSelfLink field, or simply do a GET on the factory to get a list of example service instances

```sh
$ curl -X PATCH -H "Content-type: application/json" -d '{"name":"george"}' http://localhost:8000/core/examples/94639609-e989-4ffd-ade6-0a5f2841a421
```

Service responds with:

```json
{
  "keyValues": {},
  "name": "george",
  "documentVersion": 1,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/755c582b-eef4-4e96-8450-09e7227355af",
  "documentUpdateTimeMicros": 1411077167181000
}
```

The response has the new state, but you can also do a GET on the child service, to confirm the PATCH took effect. Notice the version is now 1.

### Statistics

Xenon tracks per service instance, per operation statistics so you can determine latency, throughput, aid with debugging of your handlers. Please refer to the [REST API page](./REST-API#per-service-stats)


## Example Service Code

The example below is of a minimal service that represents a specific document (the ExampleServiceState PODO). The service implements a PUT and a PATCH handler for processing complete or partial updates to its state. Each time the state updates the framework indexes the new state, increments the version, content signature and timestamp. It also sends notifications, replicates if the service is replicated, etc

```java
public class ExampleService extends StatefulService {

    public static final String FACTORY_LINK = ServiceUriPaths.CORE + "/examples";

    /**
     * Create a default factory service that starts instances of this service on POST.
     * This method is optional, {@code FactoryService.create} can be used directly
     */
    public static FactoryService createFactory() {
        return FactoryService.createIdempotent(ExampleService.class, ExampleServiceState.class);
    }

    public static class ExampleServiceState extends ServiceDocument {
        public static final String FIELD_NAME_KEY_VALUES = "keyValues";

        public Map<String, String> keyValues = new HashMap<>();
        public Long counter;
        @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
        public String name;
    }

    public ExampleService() {
        super(ExampleServiceState.class);
        super.toggleOption(ServiceOption.PERSISTENCE, true);
        super.toggleOption(ServiceOption.REPLICATION, true);
        super.toggleOption(ServiceOption.INSTRUMENTATION, true);
        super.toggleOption(ServiceOption.OWNER_SELECTION, true);
    }

    @Override
    public void handleStart(Operation startPost) {
        if (startPost.hasBody()) {
            ExampleServiceState s = getBody(startPost);
            logFine("Initial name is %s", s.name);
        }
        startPost.complete();
    }

    @Override
    public void handlePut(Operation put) {
        ExampleServiceState newState = getBody(put);
        ExampleServiceState currentState = super.getState(put);

        // example of structural validation: check if the new state is acceptable
        if (currentState.name != null && newState.name == null) {
            put.fail(new IllegalArgumentException("name must be set"));
            return;
        }

        updateCounter(newState, currentState, false);

        // replace current state, with the body of the request, in one step
        super.setState(put, newState);
        put.complete();
    }

    @Override
    public void handlePatch(Operation patch) {
        ExampleServiceState updateState = updateState(patch);
        if (updateState == null) {
            return;
        }
        // updateState method already set the response body with the merged state
        patch.complete();
    }

    private ExampleServiceState updateState(Operation update) {
        // A Xenon service handler is state-less: Everything it needs is provided as part of the
        // of the operation. The body and latest state associated with the service are retrieved
        // below.
        ExampleServiceState body = getBody(update);
        ExampleServiceState currentState = getState(update);
        boolean hasStateChanged = Utils.mergeWithState(getStateDescription(),
                currentState, body);
        // update state

        updateCounter(body, currentState, hasStateChanged);

        if (body.keyValues != null && !body.keyValues.isEmpty()) {
            for (Entry<String, String> e : body.keyValues.entrySet()) {
                currentState.keyValues.put(e.getKey(), e.getValue());
            }
        }
        if (body.documentExpirationTimeMicros != 0) {
            currentState.documentExpirationTimeMicros = body.documentExpirationTimeMicros;
        }

        // response has latest, updated state
        update.setBody(currentState);
        return currentState;
    }

    private boolean updateCounter(ExampleServiceState body,
            ExampleServiceState currentState, boolean hasStateChanged) {
        if (body.counter != null) {
            if (currentState.counter == null) {
                currentState.counter = body.counter;
            }
            // deal with possible operation re-ordering by simply always
            // moving the counter up
            currentState.counter = Math.max(body.counter, currentState.counter);
            body.counter = currentState.counter;
            hasStateChanged = true;
        }
        return hasStateChanged;
    }

    /**
     * Provides a default instance of the service state and allows service author to specify
     * indexing and usage options, per service document property
     */
    @Override
    public ServiceDocument getDocumentTemplate() {
        ServiceDocument template = super.getDocumentTemplate();
        PropertyDescription pd = template.documentDescription.propertyDescriptions.get(
                ExampleServiceState.FIELD_NAME_KEY_VALUES);

        // instruct the index to deeply index the map
        pd.indexingOptions.add(PropertyIndexingOption.EXPAND);

        PropertyDescription pdName = template.documentDescription.propertyDescriptions.get(
                ExampleServiceState.FIELD_NAME_NAME);

        // instruct the index to enable SORT on this field.
        pdName.indexingOptions.add(PropertyIndexingOption.SORT);

        // instruct the index to only keep the most recent N versions
        template.documentDescription.versionRetentionLimit = ExampleServiceState.VERSION_RETENTION_LIMIT;
        return template;
    }
}
```

For an explanation of the options a service can declare, please see the [programming model page](./Programming-Model).

# Additional Reading

 * [overview slides](https://github.com/vmware/xenon/blob/master/contrib/docs/Xenon.pptx)
                                                                   
                                                                    * [programming model](./Programming-Model) for an overview of the Xenon API surface and model.
                                                                   
                                                                    * [service author guidelines](./service-implementation-guidelines)

 * [design patterns](./Service-Design-Patterns)

 * [debugging and troubleshooting](./Debugging-and-Troubleshooting#starting-a-host) for more details on starting Xenon, including starting multiple, replicated hosts.

 * [custom host and service creation page](./Hosting-Custom-Services-On-Xenon) if you want to create and host custom services.

 * [custom UI per service](./Host-Your-UI)

 * [developer guide](./Developer-Guide) for instructions on how to build, pre reqs, etc

 * [testing guide](./Testing-Guide) for instructions on how to easily, and locally, test on single or multiple nodes

