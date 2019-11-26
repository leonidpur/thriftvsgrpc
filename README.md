# In search of the best RPC: Thrift vs. gRPC/Protobuf

Why do we need RPC at all?
In the era of RESTful/JSON RPC might seem to be a redundant option. But is it always true?
C++ doesn’t have native reflection and annotations like other more dynamic languages. RPC's direct parsing (performed by dedicated parser from machine-generated code) is faster by factor than the parsing of REST-based systems. The later actually perform dictionary match between parsed json and reflected objects. RESTful/Json also has more overhead because of HTTP and ASCII encodings. 

So, when performance matters, you prefer RPC. If your client/server application has C++ at least on one side, you may have no other choice but to use RPC. For that reason, this comparison is focused more on the C++ version and less on Java, C#, and Python.

Thrift is a self-contained RPC stack. 
gRPC is a conjunction of gRPC and Protobuf. Protobuf is responsible for interface definition and serialization. gRPC along is for transport and procedure call. In the article, gRPC generally means gRPC/Protobuf pair.



## Table of contents
* Introduction
* Performance 
* Supported platforms
* Supported languages
* Cross-language, cross-platform
* Supported configurations
* Generated code
* Supported data models 
* Library complexity

## Intro

From wiki: “Although developed at Facebook, it is now an open source project in the Apache Software Foundation. The implementation was described in an April 2007 technical paper released by Facebook, now hosted on Apache”
“gRPC (gRPC Remote Procedure Calls[1]) is an open source remote procedure call (RPC) system initially developed at Google.”

gRPC/Protobuf is released later.

Basically, both are RPC but:
gRPC addresses more issues. This paper consists of 2 parts:
1. Close comparison of basic RPC staff which applies to both parties.
2. Concepts which are not part of “classic” RPC and applicable to only one side (mostly gRPC). 
## Supported languages
The list of Thrift-supported languages is great: https://thrift.apache.org/docs/Languages  
The list of gRpc/Proto is more modest: https://developers.google.com/protocol-buffers/docs/proto3  
Still, the most popular languages are supported by both Thift and Protobuf.

Cross-language calls are absolutely possible. When your client language is different from server’s one you need to “compile” the same interface twice for both client/server languages.

## Supported platforms
For non-C++ it naturally runs where Language’s VM/Interpreter runs. Concerning C++ Thrift, as far as I saw, any possible combination of cross-platform calls runs well: Windows or Linux, x64, x86, ARM, any Endianness. As for gRpc, I checked Windows to Linux link and it was also ok.

## Call structure
Thrift is a very idiomatic RPC call. There are multiple or none input parameters of any supported or custom type and one return value or void. It doesn't support COM-style output parameters for a simple reason just because it’s not valid in all target languages. On the other hand, Thrift supports exceptions. You can catch Thrift's exceptions and user-defined exception types.

gRPC reduced the call semantics to the “request-response” paradigm. Always one input parameter(“messages”) and one return “message”. Even if you don’t need one, you have to define “dummy”-empty one type. If the only parameter is “int”, for example, you wrap "int" in custom “message” type. As for errors, it returns error status in c++ or throws error exceptions in java, c#, python. User-type exceptions are not supported.
## Supported types
In both technologies, you can, practically, define every class you want.
Both support Primitives, Enums, Collections, Nested, e.t.c.

Both don’t: Polymorphism. It’s a common limitation for all RPC implementations. The possible workaround is to duplicate signatures to support all possible subclasses.

However, when it comes to nuances you can notice that not every variable width or encoding is supported.
https://thrift.apache.org/docs/types
https://developers.google.com/protocol-buffers/docs/proto3

It can be a serious issue for massive collections. For example, if you need to pass a big collection of small-range decimals: Thrift supports only doubles, while Protobuf both double and float. Protobuf also supports signed/unsigned and fixed-width integers. On the other hand, only Thrift supports 8-bit, 16-bit integers.

Other nice features in Protobuf having no equivalent in Thrift:
“Any” - attaching “black-box” type in messages without defining it in .proto
“Oneof” - similar to the idea of union in c++. Members of struct reuse common space when defined as “OneOf”.

## Generated code
The generated C++ or Java code would seem not exactly beautified. Particularly, generated C++ or Java may not be in line with your favorite coding style.

Thrift translates collections into c++ native stl collections.
Protobuf translates collections into its own collections. The reason: it supports more sophisticated memory models.
## Design
**Thrift**
Thrift is a self-contained suite. You can use it full-stack for RPC or just only a serialization layer. 
https://thrift.apache.org/docs/concepts
It’s a very open architecture and layered in a way similar to the networking stack. There is polymorphism in each layer and you can implement your layer's module. For example, in transport layer abstract TTransport is subclassed to TSocket, THttpTransport and TPipe.
If you have your special protocol you can subclass TTransport and use it with the rest of the framework instead of/alongside with the known subclasses.
The same story is the for Protocol layer where it has Binary, JSON, Compact e.t.c.
As an already supported transport you can choose between TCP, Http, Pipe or UNIX-Domain Socket.
Unfortunately, in the case of the Intra-machine (Local)RPC, there is no transport based on shared-memory. No transport based on ultrafast Windows ALPC.

**gRPC**
gRPC is a more concrete suite and you should use it as is. The transport is only HTTP/2-based.
It does support compression of HTTP/2 payload. As for replacing message format by non-Protobuf binary, you can decouple gRPC from Protobuf and use other formats, for example, JSON. But in this case, you’re responsible for formatting and unmarshalling. In other words, it ceases to be a full-spec RPC. Still, taking advantage of powerful gRPC transport and method dispatching.

## Internal flow
Thrift is classically built on top of Berkeley sockets. Client-side serializes the data and writes it into socket. On Server’s side blocking server’s: Dedicated thread reads data, un-serializes it and dispatches to appropriate call. Thrift’s answer to big amount of clients is "Non-blocking" server which is based on socket's “select” function.

gRPC is built on top of completion queue before it reaches sockets and on another completion queue when it’s picked from socket and processed further. Both client and server. Queued data is served by threads from the pool.
Apparently, the queued model is supposed to serve big traffic better. It also shifts load from OS space of sockets to user’s space of gRPC queue.


## Performance
In general, both are considered as best performers.

In my test, the same test framework tests “logically identical“ Thrift and gRpc calls. I measure time taken by clients-side function which:
1. packs input in client from user’s entities to machine-generated types
2. call to interface method
3. In server-side, implementation function extracts data from generated types and packs machine-generated return value
4. Client waits for return of the interface method.

In other words, it should indicate how efficient is the full-stack of client/server RPC.

I used 2 benchmarks
Getting of 1 Mb of “binary” (method A)
Setting and getting of array of structs (method B) where the “struct” is:

-----------------------------------------------------------------------
>
	std::vector<std::shared_ptr<AppSampleData>>  aggregate;
	struct AppNestedData 
	{
		int32_t int32NestedData;
		int64_t int64NestedData;
	}

	struct AppSampleData 
	{
		int32_t int32SampleData;
		int64_t int64SampleData;
		double doubleSampleData;
		AppNestedData nestedSampleData;
	}
>
--------------------------------------------------------------------

As a baseline (for measurement of system/network throughput) I used iPerf (https://iperf.fr/)

Concerning configurations, I used:
Thrift: Threaded server, Binary protocol
gRpc: One-channel client/s, default ServerBuilder, Synchronous, Unary

Two these configurations are logically equivalent.
Depending on the environment or scenario other configuration can give some up to~20% speed-up, For example:
Thrift: Pooled server, Non-blocking server or Compact protocol
gRpc: Streams, Asynchronous, Arena. It also supports multiple channels(aka connections) which has no equivalent in Thrift.

However, even comparison of these basic configurations gives some idea about pros/cons of each.
I want to measure throughput per second and standard deviation for 1000 samples. The second one should give me an idea of how predictable(or consistent) the delay is.

Benchmark 1: Same Win64 machine where iPerf measured 11.4 Gbits/sec of inter-process throughput.

| Tables   |      Thrift: <br>method A      |  gRpc:<br> method A |Thrift<br> method B |gRpc:<br> method B |
|----------|:-------------:|:------:|:------:|:------:|
| One thread connection |  12.44Gbits/sec <br>sD = 578.44 | 2.08Gbits/sec <br> sD = 3595.73 | 652.03 Mbits/sec<br> sD = 12362.37 | 450.15<br> Mbits/sec sD = 16976.56 |
| 2-threads/connections |  11.32Gbits <br>sD = 1323.59 | 4.16Gbits <br> sD = 3683.53 | 1.09 Gbits/sec<br> sD = 12954.04 | 873.16 Mbits/sec<br> sD = 17505.14 |
| 5-threads/connections | 10.68 sD = 3701.62 | 4.26 sD = 5347.03 | 1.96 Gbits/sec sD = 19042.23 | 1.44 Gbits/sec sD = 25893.98 |
| 10--threads/connections | 9.32 sD = 8329.04 | 2.18 Gbits/sec sD = 42176.65 | 3.35 Gbits/sec sD = 22947.09 | 1.44 Gbits/sec sD = 25893.98 |























One can see that Thrift is significantly faster and remarkably more consistent(lower standard deviation). However, for complex types (method B) the gap is not as big as for plain data (method A). The possible reason is that type-manipulation of Protobuf-generated types is more efficient.

Benchmark 2: Two Win64 machines where iPerf measured 95.0 Mbits/sec inter-process throughput




Thrift: Retrieving byte array
1 Mb (500 samples) 
gRpc: Retrieving byte array
1 Mb (1000 samples) 
Thrift exch
grp
One thread connection


89.31 MBits/sec
sD = 85435.05
2.08
sD = 3595.73
652.03 Mbits/sec
sD = 12362.37
450.15 Mbits/sec
sD = 16976.56
2-threads/connections



90.6 MBits/sec
sD = 168410.95
4.16
sD = 3683.53
1.09 Gbits/sec
sD = 12954.04
873.16 Mbits/sec
sD = 17505.14
5-threads/connections


90.4 
sD = 420918.28
4.26
sD = 5347.03
1.96 Gbits/sec
sD = 19042.23
1.44 Gbits/sec
sD = 25893.98
10--threads/connections
9.32
sD = 8329.04


2.18 Gbits/sec
sD = 42176.65
3.35 Gbits/sec
sD = 22947.09














When over-socket, you can expect some 15% penalty against ideally implemented socket-based client-to-server communication of plain buffer of size compatible of the size of serialized objects. For local-RPC, pipe-based transport can perform slightly better for easy contention cases.



Why Thrift performed considerably better for simple models? Without diving deep into profiling I rushed to blame 2 factors:
1. Tcp-binary against Http/2
2. Straightforward on-top-of-socket against queueing

For more complicated models gRpc can win due to load-balancing and asynchronous callbacks.

## Load balancing
Thrift has no build-in support for LB. The connection is established once by client against concrete server point. The way to distribute calls among servers is to reconnect against LB proxy.

gRPC client (channel) is designed as a wrapper of potentially multiple connections to multiple server points. Load balancing occurs per each call.


## Special technologies
gRPC is more than idiomatic RPC. Examples of thechnologines absent in Thrift are:

**Streams** - Similar to Websocket’s opposite traffic. Server can return not just concrete data but a “stream”. Client can read the stream as long as Server sends new data. Thrift has no such“callbacks”. If you Thrift’s server has data to push you should create opposite Server on Client-side

**gRPC/Protobuf Arena** - reuse of memory from pool for frequent object creations

**gRPC call cancelation** - Client can terminate request explicitly or by timeout untill it has returned. 

## Conclusion
Which one to choose?

*Thrift* looks preferable when you:
* Look for mature, time-tested idiomatic RPC
* May overwrite specific components if you have custom transport, encoding e.t.c
* Have model of limited scale and look for the best performer (somewhat sort real-time) for that concrete model
* You need to economize resources (Embedded platform)
* Your language is not supported by gRPC/Protobuf (For example, Rust)

Choose *gRpc/Protobuf* if:
* Your model is scalable (think load-balancing)
* You have data pushes from servers (think streams)
* You have discovered important primitive types, which are not supported by Thrift in the best way
* Your custom data types are over-complicated and you want memory optimization (Arena)
* You want to cancel long calls

Best wishes!
