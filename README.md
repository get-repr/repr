# Remote Execution Project (repr)

This repository contains the core client and server applications for the Remote Execution Project, abbreviated repr (pronouned "reaper").

Repr is a project to enable remote execution of processes across the network. The core motivations of this project are to make remote execution as seamless and scalable as possible.

Repr currently only supports Windows.

## Core Design Philosophies

Repr has a number of core design philosophies that drive the technical design.

### Virtualized Environments

The environment in which work is being remotely executed should be virtualized as much as possible. This means that any resources that the work needs to access (such as reading files) should be intercepted and the request sent back to the requester.

This allows the requester to not worry about shuffling around the resources used by the process to make it portable.

### Scalable Execution

Repr should allow you to scale remote execution to many environments and continue to have it be performant. Specifically, we should minimize poorly scaling load on the requester. A concrete example would be for the requester to not have to upload the same file to every remote environment.

### Extensibility

We expect that more often than not repr will be introduced to existing systems, rather than something being designed around it. Repr should be extensible, so that its functionality can be extended to fit different use cases.

## Architecture

### Overview

Repr consists of two primary applications: the client and the server.

The client is the one making remote execution requests, and the server accepts these requests and carries them out. The client and server communicate via gRPC initiated from the client. A single gRPC connection can be maintained over multiple requests for remote work. The gRPC endpoint is a bidirectional stream, so that the client can keep requesting work and server can keep requesting files or other resources.

One of the unique aspects of the design that facilitates scalability is the use of a CAS as shared storage between the client and server. The CAS allows for speedups when a client is requesting work from multiple servers, as well as when repeatedly requesting work from the same server. In cases where the CAS is shared between different clients, there are additional opportunities for efficiency.

As there is no shared storage mechanism that can be assumed to be available, Repr does not come with any CAS implementations. Instead plugins must be used to define the specific CAS behavior desired.

### Plugins

Repr uses plugins to extend and customize its behavior. One of the most notable plugin interfaces is the CAS implementation, which facilitates the sharing of files between the client and server.

Plugins can facilitate many other kinds of behavior as well, such as telemetry + operational monitoring, repr server discovery, authentication, build system compatibility + interoperability, and more.

### Server

The repr server is a service that accepts and carries out remote execution requests. The server consists of 3 components: a minifilter driver, a ProjFS implementation, and a gRPC service.

Note: The gRPC service and ProjFS implementation will likely live in the same runtime, but can be thought of as logically distinct parts of the server.

The gRPC service is what interacts with the client, as well as the CAS or other services via plugins. When receiving a request, the service will then configure the minifilter driver and ProjFS implementation before starting the requested work.

When work is started, the first server component that will be hit is the minifilter driver. The minifilter will recognize any file requests from this process (or its children) and redirect the requests to a work request-specific staging directory. For example, if the process requests `C:\a\b.txt`, the minifilter may redirect this to a directory like `C:\repr\work\request0001\C\a\b.txt`.

At this point, the IO will be handed off to ProjFS. The ProjFS implementation will have been set up to virtualize the entire staging for the request. In the example above, the `C:\repr\work\request0001` directory would be virtualized. This allows the ProjFS implementation to populate those files based on files received from the client. It does this be handing these requests to the gRPC service implementation.

When the gRPC service receives a file request from the ProjFS, it sends a message over the established connection to the client (remember, the gRPC endpoint is bidirectional) to retrieve the file contents. The client will then upload the file to the CAS (or confirm that it's already there) and send back a message with the CAS key for that file. The service implemenetation will then download the file from the CAS (potentially with additional caching layers) and hand that data to the ProjFS implementation, which in turn will write it to the ProjFS backend. At this point, the original file IO request is unblocked and the work process can continue.

### Performance Improvements

In our opinion, the desire to remotely execute work is a statement about prioritizing scalability and throughput over latency, and the design of repr reflects that. That being said, there are mechanisms that can be used to improve all around performance.

One of these is through pre-populated CAS mappings. If the client knows the file dependencies of the work being done ahead of time, it can upload those files to the CAS and populate the path -> CAS key mappings in the initial requests to the server, so that there is no additional round-trip cost to requesting those files. Similarly, plugins can be used to start downloading those files to a local cache so that later ProjFS requests can be served immediately. Telemetry plugins can be used to analyze which files are most frequently being missed or are costing you the most time.

Of course, these mechanisms are still backed by the fully asynchronous flows laid out above, so even a "best guess" mapping by the client can give you performance without costing reliability.
