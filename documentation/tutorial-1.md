---
title: "Single-machine \"Replicable\" test-drive"
last_updated: August 18, 2016
sidebar: documentation_sidebar
toc: false
---

## The `gigapaxos.properties` file

The default gigapaxos.properties file in the top-level directory has two sets of entries respectively for "active" replica servers and "reconfigurator" servers. Every line in the former starts with the string `active.` followed by a string that is the name of that active server (e.g., `100, 101 or 102` below) followed by the separator '=' and a `host:port` address listening address for that server. Likewise, every line in the latter starts with the string `reconfigurator.` followed by the separator and its ` host:port` information

The `APPLICATION` parameter below specifies which application we will be using. The default is `edu.umass.cs.reconfiguration.examples.noopsimple.NoopApp`, so uncomment the `APPLICATION` line below (by removing the leading `#`) as we will be using this simpler "non-reconfigurable" `NooPaxosApp` application in this first tutorial. A non-reconfigurable application's replicas can not be moved around by gigapaxos, but a `Reconfigurable` application (such as `NoopApp`) replicas can.

```
#APPLICATION=edu.umass.cs.gigapaxos.examples.noop.NoopPaxosApp
    
active.100=127.0.0.1:2000
active.101=127.0.0.1:2001
active.102=127.0.0.1:2002

reconfigurator.RC0=127.0.0.1:3100
reconfigurator.RC1=127.0.0.1:3101
reconfigurator.RC2=127.0.0.1:3102
```

At least one active server is needed to use gigapaxos. Three or more active servers are needed in order to tolerate a single active server failure. At least one reconfigurator is needed in order to be able to reconfigure RSMs on active servers, and three or more for tolerating reconfigurator server failures. Both actives and reconfigurators use consensus, so they need at least 2f+1 replicas in order to make progress despite up to f failures. Reconfigurators form the "control plane" of giagpaxos while actives form the "data plane" that is responsible for executing client requests.

For the single-machine, local test, except for setting `APPLICATION` to `NoopPaxosApp`, you can leave the default gigapaxos.properties file as above unchanged with 3 actives and 3 reconfigurators even though we won't really be using the reconfigurators at all. 

## Starting and stopping servers

Run the servers as follows from the top-level directory:

```
./bin/gpServer.sh start all
```

If any actives or reconfigurators or other servers are already listening on those ports, you will see errors in the log file (` /tmp/gigapaxos.log `) by default). To make sure that no servers are already running, do

```
./bin/gpServer.sh stop all
```

To start or stop a specific active or reconfigurator, replace `all` above with the name of an active (e.g., `100`) or reconfigurator (e.g., `RC1`) above.

Wait until you see `all servers ready` on the console before starting any clients.

## Starting the client

Then, start the default client as follows from the top-level directory:

```
./bin/gpClient.sh
```

The client will by default use `NoopPaxosAppClient` if the application is `NoopPaxosApp`, and will use `NoopAppClient` if the application is the default `NoopApp`. As we are using the former app in this tutorial, running the above script will launch `NoopPaxosAppClient`.

For any application, a default paxos group called `\<app_name\>0` will be created by the servers, so in this example, our (only) paxos group will be called `NoopPaxosApp0`. 

The `NoopPaxosAppClient` client will simply send a few requests to the servers, wait for the responses, and print them on the console. The client is really simple and illustrates how to send callback-based requests. You can view its source here: [NoopPaxosClient.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/gigapaxos/examples/noop/NoopPaxosAppClient.java>)

`NoopPaxosApp` is a trivial instantiation of `Replicable` and its source is here: [NoopPaxosApp.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/gigapaxos/examples/noop/NoopPaxosApp.java>)

You can verify that stopping one of the actives as follows will not affect the system's liveness, however, any requests going to the failed server will of course not get responses. The `sendRequest` method in `NoopPaxosAppClient` by default sends each request to a random replica, so roughly a third of the requests will be lost with a single failure. 

```
bin/gpServer.sh stop 101
```

Next, browse through the methods in `NoopPaxosAppClient`'s parent [PaxosClientAsync.java](<https://github.com/MobilityFirst/gigapaxos/blob/master/src/edu/umass/cs/gigapaxos/PaxosClientAsync.java>) and use one of the `sendRequest` methods therein to direct all requests to a specific active server and verify that all requests (almost always) succeed despite a single active failure. You can also verify that with two failures, no requests will succeed.
