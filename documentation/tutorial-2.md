---
title: "Single-machine \"Reconfigurable\" test-drive"
last_updated: August 18, 2016
sidebar: documentation_sidebar
toc: false
---

For this test, we will use a fresh gigapaxos install and set `APPLICATION` to the default `NoopApp` by simply re-commenting that line as in the default gigapaxos.properties file as shown below.

```
#APPLICATION=edu.umass.cs.gigapaxos.examples.noop.NoopPaxosApp  
#APPLICATION=edu.umass.cs.reconfiguration.examples.noopsimple.NoopApp # default
```

Note: It is important to use a fresh gigapaxos install as the `APPLICATION` can not be changed midway in an existing gigapaxos directory; doing so will lead to errors as gigapaxos will try to feed requests to the application that the application will fail to parse. An alternative to a fresh install is to remove all gigapaxos logs as follows from their default locations (or from their non-default locations if you changed them in gigapaxos.properties):

```
rm -rf ./paxos_logs ./reconfiguration_DB
```

Next, run the servers and clients exactly as before. You will see console output showing that `NoopAppClient` creates a few names and successfully sends a few requests to them. A `Reconfigurable` application must implement slightly different semantics from just a `Replicable` application. You can browse through the source of `NoopApp` and `NoopAppClient` and the documentation therein below:

[NoopApp.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/noopsimple/NoopApp.java>) 
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/noopsimple/NoopApp.html>)

[NoopAppClient.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/NoopAppClient.java>)
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/NoopAppClient.html>)

Step 1: Repeat the same failure scenario as above and verify that the actives exhibit the same liveness properties as before.

Step 2: Set the property RECONFIGURE_IN_PLACE=true in gigapaxos.properties in order to enable *trivial* reconfiguration, which means reconfiguring a replica group to the same replica group while going through all of the motions of the three-phase reconfiguration protocol (i.e., `STOP` the previous epoch at the old replica group, `START` the new epoch in the new replica group after having them fetch the final epoch state from the old epoch's replica group, and finally having the old replica group `DROP` all state from the previous epoch). 

The default reconfiguration policy trivially reconfigures the replica group after *every* request. This policy is clearly an overkill as the overhead of reconfiguration will typically be much higher than processing a single application request (but it allows us to potentially create a new replica at every new location from near which even a single client request originates). Our goal here is to just test a proof-of-concept and understand how to implement other more practical policies.

Step 3: Run `NoopAppClient` by simply invoking the client command like before:

```
bin/gpClient.sh
```

`NoopApp` should print console output upon every reconfiguration when its `restore` method will be called with a `null` argument to wipe out state corresponding to the current epoch and again immediately after when it is initialized with the state corresponding to the next epoch for the service name being reconfigured.

Step 4: Inspect the default *reconfiguration policy* in
[DemandProfile.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/reconfigurationutils/DemandProfile.java>) [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/reconfigurationutils/DemandProfile.html>) 
and the abstract class 
[AbstractDemandProfile.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/reconfigurationutils/AbstractDemandProfile.java>)
[[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/reconfigurationutils/AbstractDemandProfile.java>)
that any application-specific reconfiguration policy is expected to
extend in order to achieve its reconfiguration goals.

Change the default reconfiguration policy in `DemandProfile` so that the default service name `NoopApp0` is reconfigured less often. For example, you can set `MIN_REQUESTS_BEFORE_RECONFIGURATION` and/or `MIN_RECONFIGURATION_INTERVAL` to higher values. There are two ways to do this: (i) the quick and dirty way is to change `DemandProfile.java` directly and recompile gigapaxos from source; (ii) the cleaner and recommended way is to write your own policy implementation, say `MyDemandProfile`, that either extends `DemandProfile` or extends
`AbstractDemandProfile` directly and specify it in gigapaxos.properties by setting the `DEMAND_PROFILE_TYPE` property by uncommenting the corresponding line and replacing the value with the canonical class name of your demand profile
implementation as shown below. With this latter approach, you just need the gigapaxos binaries and don't have to recompile it from source. You do need to compile and generate the class file(s) for your policy implementation.

```
#DEMAND_PROFILE_TYPE=edu.umass.cs.reconfiguration.reconfigurationutils.DemandProfile
```

If all goes well, with the above changes, you should see `NoopApp` reconfiguring itself less frequently as per the specification in your reconfiguration policy!

Troubleshooting tips: If you run into errors:

(1) Make sure the canonical class name of your policy class is correctly specified in gigapaxos.properties and the class exists in your classpath. If the simple policy change above works as expected by directly modifying the default `DemandProfile` implementation and recompiling gigapaxos from source, but with your own demand profile implementation you get `ClassNotFoundException` or other runtime errors, the most likely reason is that the JVM can not find your policy class.

(2) Make sure that all three constructors of DemandProfile that respectively take a `DemandProfile`, `String`, and `JSONObject` are overridden with the corresponding default implementation that simply invokes `super(arg)`; all three constructors are necessary for gigapaxos' reflection-based demand profile instance creation to work correctly.

Step 5: Inspect the code in
[NoopAppClient.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/NoopAppClient.java>)
[[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/NoopAppClient.html>)
to see how it is creating a service name by sending a `CREATE_SERVICE_NAME` request. A service name corresponds to an RSM, but note that there is no API to specify the set of active replicas that should manage the RSM for the name being created. This is because gigapaxos randomly chooses the initial replica group for each service at creation time. Applications are expected to reconfigure the replica group as needed after creation by using a policy class as described above.

Once a service has been created, application requests can be sent to it also using one of the `sendRequest` methods as exemplified in `NoopAppClient`.

Deleting a service is as simple as issuing a `DELETE_SERVICE_NAME` request using the same `sendRequest` API as `CREATE_SERVICE_NAME` above.

Note that unlike `NoopPaxosAppClient` above, `NoopAppClient` as well as the corresponding app, `NoopApp` use a different request type called `AppRequest` as opposed to the default `RequestPacket` packet type. Reconfigurable gigapaxos applications can define their own extensive request types as needed for different types of requests. The set of request types that an application processes is conveyed to gigapaxos via the `Replicable.getRequestTypes()` that the application needs to implement.

Applications can also specify whether a request should be paxos-coordinated or served locally by an active replica. By default, all requests are executed locally unless the request is of type `ReplicableRequest` and its `needsCoordination` method is true (as is the case for `AppRequest` by default).

Verify that you can create, send application requests to, and delete a new service using the methods above. 

A list of all relevant classes for Tutorial 2 mentioned above is listed below for convenience:

[NoopAppClient.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/NoopAppClient.java>)
[[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/NoopAppClient.html>)

[NoopApp.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/noopsimple/NoopApp.java>)
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/noopsimple/NoopApp.html>)

[AppRequest.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/examples/AppRequest.java>)
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/examples/AppRequest.html>)

[ReplicableRequest.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/interfaces/ReplicableRequest.java>)
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/interfaces/ReplicableRequest.html>)

[ClientRequest.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/gigapaxos/interfaces/ClientRequest.java>)
 [[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/gigapaxos/interfaces/ClientRequest.html>)

[ReconfigurableAppClientAsync.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/reconfiguration/ReconfigurableAppClientAsync.java>)
[[doc]](<https://mobilityfirst.github.io/gigapaxos/doc/edu/umass/cs/reconfiguration/ReconfigurableAppClientAsync.html>)
