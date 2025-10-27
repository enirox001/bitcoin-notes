Given an interface description of an object that has one or more methods, libmultiprocess generates the 
1. C++ Proxy-client which is a class template specialization with an implementation of each of the interface method, that sends over a socket, waits for the response and returns the result.
2. C++ Proxy-server which is a class template specialization that listens for requests over a socket and calls a wrapped C++ object that implements the same interface to execute the request.
```
   Proxy Client -> Proxy Server -> Implementation of Interface
```

The function call to request translation supports input and output arguments, such that standard types like `unique_ptr` `vector` `map` and `optional`, it also supports bidirectional calls between process through interface pointer and `std::function` arguments.

If a wrapped C++ object inherits from an abstract base class that declares the virtual methods (this is different from the Proxy-server, the wrapped C++ object is an object that implements the same interface), the generated Proxy-client object can inherit from the same class, and this would allow inter process calls to replace local calls with no change to existing code.

An optional support for thread mapping also exists such that each thread making inter process calls can have a dedicated thread processing requests from it, callbacks from processing threads are executed on corresponding request threads (threads for processing requests and corresponding request threads)

libmultiprocess is a pure wrapper over the protocol so that clients and servers written in other languages but using the capnproto schema can communicate with inter process counter-parties using libmultiprocess without using or knowing the details of libmultiprocess

## Core Architecture
