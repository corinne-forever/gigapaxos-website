---
title: "Getting Started"
last_updated: August 18, 2016
sidebar: documentation_sidebar
toc: false
---

## Obtaining
Prerequisites: `JRE1.8+`, `bash`, `ant` (source only)

Binary:

* Download the latest stable binary [here](https://github.com/MobilityFirst/gigapaxos/releases).

Source:

* Clone the GigaPaxos repo from [Github](https://github.com/MobilityFirst/gigapaxos)
* In the main directory, type `ant`, which will create a jar file   `dist/gigapaxos-\<version\>.jar`. Make sure that `ant` uses java1.8 or higher.

## Using GigaPaxos 
GigaPaxos has a simple `Replicable` wrapper API that any “black-box” application can implement in order for it to be automatically be replicated and reconfigured as needed by GigaPaxos. This API requires three methods to be implemented: 

```
boolean execute(Request request) 
String checkpoint(String name)
boolean restore(String name, String state) 
```

to respectively execute a request, obtain a state checkpoint, or to roll back the state of a service named “name”. GigaPaxos ensures that applications implementing the `Replicable` interface are also automatically `Reconfigurable`, i.e., their replica locations are automatically changed in accordance with an application-specified policy.

From here you should checkout the [first tutorial]({{ site.baseurl }}/documentation/tutorial-1/).
