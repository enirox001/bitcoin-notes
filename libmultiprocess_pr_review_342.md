_my thoughts are in italics, when i can put them_
### issue
libmultiprocess requests are blolcing, with no way to cancel them from either the client or server side, downstram works around this by pairing each blocking method with a dedicated interruptX method whose only pupose is to wake it up

### pr 342
**allow request cancellation for wrapped c++ methods** [link](https://github.com/bitcoin-core/libmultiprocess/pull/342)

#### desc
capnp provides a useful cancelation mechanism that multiprocess can use to support cancellation from both non libmultiprocess and libmultiprocess clients. When a promise is dropped, it sends a cancellation request, the server can cancel immediately 
the pr implements this b y adding cancellation support on both side
- allow `ProxyClient` method calls to be cancelled from another thread through a cancel function, this is backed by `kj::Canceler` that wraps the request promise and is attached to it. A canceled call throws `InterruptException`
- Allow wrapped server methods to register a callback that runs when the request is canceled wether because the promise dropped or the connection was interrupted
the pr also adds a new capnp annotation, `$Proxy.extraParam`, which declares c++ only parameters that are not sent through rpc, these parameters are handled by `CustomBuildExtraParam` on the client side and `CustomReadExtraParam` on the server side, the first user of this mechanism is the cancellation parameter type `std::function<void(std::function<void()>)`

this capnp method 
```
waitNext @0 (context :Proxy.Context) -> (result :Template) $Proxy.extraParam("cancel") $Cxx.allowCancellation;
```
maps to a c++ method with a trailing cancellation argument, without requiring any libmultiprocess type in the interface header
```
using CancelFn = std::function<void()>;
using CancelArg = std::function<void(CancelFn)>;

virtual std::unique_ptr<Template> waitNext(CancelArg cancel) = 0;
```
on the client, the argument receives a function that can cancel the call
```
CancelFn cancel_fn;

// ... blocks until the result arrives or another thread runs cancel_fn()
auto tmpl = client->waitNext([&](CancelFn fn) {
    cancel_fn = std::move(fn);
});
```
on the server the implementation registers a callback that interrupts its wait:
```
std::unique_ptr<Template> waitNext(CancelArg cancel) override
{
    if (cancel) cancel([this] { m_cv.notify_all(); });

    // ... wait on m_cv, checking for cancellation
}
```
detecting a dropped promise on the server requires the `$Cxx.allowCancellation` annotation in the method, file or interface, without it capnp runs the abandoned call to completion

_thoughts_
this pr seems to let one thread stop and ipc call that another thread is blocked waitingfor, without disconnecting or breaking the whole ipc connection

it works by
- the client starting a blocking request and receives a cancel function
- another thread can call that function
- the waiting client wakes up and gets an `InterruptException`
- the server is notified and runs a callback registered by the server method
- that callback must cooperatively stop the server's work, it does not forcibly kill the server thread
- the connection remains usable for future calls

a new capnp param is also introduced, This adds a c++ only argument to a generated method wihout putting that argument into the network message, the Cancellation uses this mechanism so other capnp clients do not see the new field

the cancellation function itself is not sent to the server, capnp sends the cancellation signal; each side creates its own local callback around that signal

this is somewaht inline with the pr we reviewed in 335 that drains and disconnects, with this change it would make the connection object be preserved even when the disconnection happens on one thread or another.

#### Comments so far
ryanofsky commended the work
- did not think the capnp cancel struct should exist, the idea of allowing a cancelToken/argument is just to procide a way for multiprocess c++ clients to cancel `capnp::Response` promises, and for the servers to detect that they have been cancelled, 
	- xyz responded that this is better and would have benefits
- Relatedly the implementation should be orthogonal to the allowCancellation annotation, the annotation controls how `capnp` sends cancellations, while `CancelToken` is way of letting libmultiprocess classes send and receive them. It will be more useful combined with the allowCancelation annotations
	- xyz agrees this is better as well as the allowCancellation only changes how a server responds to a cacnel signal, this means the `dispatchCall` override can be dropped
- in order for this function to be usefil in bitcoin, it should not require c++ interfaces to directly use the `CancelToken` type, Bitcoin Core C++ interfaces are intended to be used by the node and compile without any dependency on multiprocess, So multiprocess could provide `CustomBuild/ReadCancel` overloads analagous to others already existing taht appls can override to work with custom cancellation arguments. 
	- xyz said it would be better to combine with onCancel so it registers and unregisters the callback at the rigth time
		- ryan mentioend that unregistering is needed to deal with cancellation being sent of the method finishes executing, so it is pretty important - added the cancellation
- Alternately instead of adding CustomBuild/ReadCancel hooks we could use the existing CustomBuildField/CustomReadFields hooks with empty output/input args, this might need a new hook that returns true by default but false if there is no capnp field correspoding to the c++ type, this owuld allow the clientInvoke/serverInvoke to handlet c++ arguments taht dont have corresponding capnp fields
	- xyz mentioned this is prefered over the dedicated hooks but does not know how this would handle the client side, he does nto see how it makes mpgen print a parameter the schema doesnt mention
		- ryan said that the proposed plan he mentioned would ot work as code generator also needs to know the field mapping and doesnt have access to anything but the capnp files, so a capnp method annotation of some kind might be necessary
- it looks likes the implementation of https://github.com/bitcoin-core/libmultiprocess/commit/0986a13ea66a4fab1a8f64de5d37815efaaa1afa only alows servers to detect the cancellation, but does not allow clients to request it, it would make sense to support both
	- will cover both sides in this PR
- fine to drop support for older versions
- ci failures can be fixed