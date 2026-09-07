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
			- xyz mentioned that the vector of callbacks exis because another acllback is registered in `type-context.h` to log the request was cancelled mid execution and lock the `request_mutex` so the event loop cannot delet ehte params/results while the worker thead is stillusing them
			- so during a cancelable request, there are two registrations one is internal and another the wrapped method itself registers, 
- Alternately instead of adding CustomBuild/ReadCancel hooks we could use the existing CustomBuildField/CustomReadFields hooks with empty output/input args, this might need a new hook that returns true by default but false if there is no capnp field correspoding to the c++ type, this owuld allow the clientInvoke/serverInvoke to handlet c++ arguments taht dont have corresponding capnp fields
	- xyz mentioned this is prefered over the dedicated hooks but does not know how this would handle the client side, he does nto see how it makes mpgen print a parameter the schema doesnt mention
		- ryan said that the proposed plan he mentioned would ot work as code generator also needs to know the field mapping and doesnt have access to anything but the capnp files, so a capnp method annotation of some kind might be necessary
- it looks likes the implementation of https://github.com/bitcoin-core/libmultiprocess/commit/0986a13ea66a4fab1a8f64de5d37815efaaa1afa only alows servers to detect the cancellation, but does not allow clients to request it, it would make sense to support both
	- will cover both sides in this PR
- fine to drop support for older versions
- ci failures can be fixed

after a look ryan aksed why the later changes use `shared_ptr` and why the cses are newly introduced instead of using a aplain `std::function` variables
- he then says the see hees that the server side `shared_ptr` was removed after an update, but the client side `shared_ptr` is still needed because when making the ipc call client side, we give the caller a void function which is copyable therefore reference counting is needed to determine the lifetime of the `kj::Canceller` object embedded in the function above, he says this could be avoided by defining the CancelFn with the operator and destructor. So that the reference is passed to CancelFn instead of a copy, this would aboid the need for reference counting

Concept ACK 

IIUC the changes here would allow one thread to stop and ipc call that another thread is blocked waiting for without disconnecting or breaking the whole ipc connection. Like that this change is somewhat inline with the changes made in #335 which would reduce the footprint for unexpected errors and behaviour in bitcoin core

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

### commit by commit review
#### [Allow wrapped C++ methods to take parameters that are not sent over RPC](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/1eaa50151680e7ebb20532c95c22c25f4593df59)

**commit mesasge**
- [x] add a `Proxy.extraParam` method annotation that declares an extra c++ only paramater in the generated method signature
	- [x] the parameter has no corresponding capnp parameter
	- [x] is no serialized or sent over rpc
	- [x] the annotation valie names the parameter in generated c++ code
- [x] client behaviour states a matching overload to the CustomBuildExtraParam must be implemented, the paremeter is passed to it befor ethe RPC message is dispatched
- [x] server behaviour states a matching CustomReadExtraParams overload must be implemented. No data arrives for this parameter so this overload reconstructs the parameter value on the server side
- [x] the test checks the values the client passes reach te client overload and the ones the server reconstructs arrive instead

*thoughts*
this commita adds a ay to say this c++ method takes an extra argument, but that argument stays locat at it isnt sent to the other ptocess

this makes it possibesutch taht when the client sends an extra arg, it only sends that value and not the extra arhg, the sever can also create its own value instead of geting the one that should be in the client side. 

This is needed because the cancelation callback is meaningful inside its own process, the client needs a function it can call to cancel the request, the server needs a function it can use toregister the response cancellation as well. Those are different lcoal functions associated with the same request, this first commit makes room for them in the c++ method signature

the tests seems good as well

i see one limitation in the argument handling,

the first one is that the server passes everya argument as an lvalue, as an existingnames variable. 

```@@ -550,9 +599,13 @@ struct ServerCall
         if (server_context.cancel_lock) server_context.cancel_lock->m_lock.unlock();
         return TryFinally(
             [&]() -> decltype(auto) {
-                return ProxyServerMethodTraits<
-                    typename decltype(server_context.call_context.getParams())::Reads
-                >::invoke(server_context, std::forward<Args>(args)...);
+                return std::apply(
+                    [&](RemoveCvRef<Extra>&... extra_args) -> decltype(auto) {
+                        return ProxyServerMethodTraits<
+                            typename decltype(server_context.call_context.getParams())::Reads
+                        >::invoke(server_context, std::forward<Args>(args)..., extra_args...);
+                    },
+                    extra);
             },
```

The ordinary arguments get forwareded but extra args do not, that means an extra value passed by value gets copied, for an integer that can be fine, but might introduce problems for more complex types.

I am thinking of forwarding based on the original extra parameter type? preserving reference parameters and allowing value parameters to move

_oops this was addressed in cfdcb45, so i guess it is more of a commit order problem_

Secondly is how the generator identifies an extra parameter, it uses a non empty name as a marker for an extra parameter, but if there is an extra entry even for `$Proxy.extraParam("")` that entry has no real capnp field, but the test treats it as an ordinary field, because its name is empty, it then attemps to obtain a name form the non extistent field. We should perhaps be rejecting empty names with a clear error

Commit ACK

#### [proxy: rename `cancel_lock` and `cancel_mutex` to `request_lock` and `request_mutex`](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/7f3199674a225215f7412470e52628461e909ef5)

**commit message**
- [x] the mutex guards the request's params and results struct not the cancellation itself
- [x] the old names would be confusing next to the `CancelState` class added in the following commits
- [x] pure rename, no behaviour change

_thoughts_
this is  a pure rename change cancel_mutex -> request_mutex and cancel_lock -> request_lock. These protext the request's input and output data when two threads interact with it.

For example a worker thread might be reading the request's arguments when cancellation arrives. The event loop thread wants to clean up the canceled request, but it must wait for the worker to finish accessing that data. The mutex and the lock coordinate this so the data isn't deleted while it is being used 

The previous name would have been confusing as it could suggest taht it controls whether cancellation happens, the new change  better describes what it protects, which is the request data.

It also prepared for the later commits which introduce actual cancelation control object. Clearer names helps to distingush their responsibilities.

this is a rename only change, and the rationale behind the changes here is sound, one thing is that there is a dangling comment that does not have the change 

```diff --git a/include/mp/type-context.h b/include/mp/type-context.h
index 63b3fdb..c1ca04f 100644
--- a/include/mp/type-context.h
+++ b/include/mp/type-context.h
@@ -154,7 +154,7 @@ auto PassField(Priority<1>, TypeList<>, ServerContext& server_context, const Fn&
         // makes another IPC call), so avoid modifying the map.
         const bool erase_thread{inserted};
         KJ_DEFER(
-            // Release the cancel lock before calling loop->sync and
+            // Release the request lock before calling loop->sync and
             // waiting for the event loop thread, because if a
             // cancellation happened, it needs to run the on_cancel
             // callback above. It's safe to release request_lock at
```

would request that this be changed

Code review ACK

#### [proxy: make client IPC calls cancelable](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/bf8a8b198e3c605fd85939309db158905a74aaf3)

**commit message**
- [x] Add `ClientCancelState` and `RequestCanceler`
- [x] `ClientCancelState` is created by `clientInvoke`
	- [x] tracks whether the call was canceled, 
	- [x] and can canel the request promise from any thread
	- [x] `RequestCanceler` inherits from `kj::Canceler`
		- [x] wraps the request promise
		- [x] is attched to it
- [x] Canceling rejects the wrapped promise
	- [x] wakes the blocked client thread through the exception path
	- [x] makes the call throw Interrupt Exception

_thoughts_
this commit adds the internal machinery for stopping a client's wait for an ipc response. Normally a client sends a request and its calling thread waits for the response, this commit wraps that pending response called a promise in a n object that can cancel it

The `CleintCancelState` keeps track of wether cancellation was requested, it can be accessed from different threads, with a mutex protecting its data

The `RequestCanceler` handles the actual cancellation of the promise. It lives on the event loop thread, where the operations run

when a cancelation is triggered, the intended sequence is

- records that the cancelation was requested
- ask the event loop thread to cancel the pending promise
- the promise's error handler makes the call as finished and wakes the waiting client thread
- the clien calls throws the Exception instead of returning a result
This cancels the individula request without closing the connection.

The commit also manages object lifetimes, it clears the pointer and the canceler when the request finishes.

i do not see much problems here, but i think there is somethings that cna be improved. 

Every IPC call now creates a `ClientCancelState` allocates a `RequestCanceler` and wraps the response promise even when the caller does not want cancellation, i think with the next commit that connects `cancel_receiver`, we could have a way where this would happen only when a cancellation receiver is present. This would reduce the allocations for ordinary calls

This cancelation and completion logic deserves more tests, for example
- cancelation is requested just as a successful response comes in
- two threads requesting cancellation
- a saved cacnelation function is called after the request finishes

I think this is better to be addressed in the last commit that introduces tests, but it would be beneficial to have at least one of these tests

Commit ACK
#### [proxy: support cancellation extra parameters](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/cfdcb45fbc616719ff4b312f66f76c6b7e27ce28)

**commit message**
- [ ] add `type-cancel.h` which defines the cancellation argument types and their extra parameter overloads
- [ ] on the client side any  `std::function<void(std::function<void()>)>` declared with `$Proxy.extraParam` receives a function that cancels the request
- [ ] on the server side, it registers a callback to run when cancellation is detected

_thoughts_
this commit connects the cancellation logic to the c++ methods, so the calles can actually use it. it links the earlier commits together to enable the functionality

on the client side a function is passed that receives a cancellation function, once the request has been sent, the library hands an `fn` calling it cancels the pending request and wakes the waiting client through `InterrupException`

on the server side, the method receives its own local registration fucntion, when the library detects cancellation, it calles taht registered callback

there are some questions i have about this commit

the cleanup in type.h happens after the wrapped method returns

```
server_context.request_lock->m_lock.lock();

// and then later on 
server_context.cancel_fn = nullptr;
```
i am considering a server method that registers a callback referring to a local mutex or a condition variable, and then finishes normally. The sequence can be

- the method destroys its local variables as it returns
- its cancellation callback is still registered
- cancellation arrives on the event loop thread before the wrapper clears that callback
- the callback accesses the destroyed variables

the request mutex protects the wrappers cleanup but it isnt held while the method destroys its locals, the later tests use callbacks capturing local variables, although their methods wait specifically for cancellation so they do not test the normal completion scenario

Do we need a way to unregister the calback before the captured variables are destroyed?

secondly the new code invokes application supplied code here 

```
invoke_context->cancel_receiver(
      [cancel_state] { cancel_state->cancel(); });
```

at that point, the request has already been sent, its response handlers capture local variables in `clientInvoke` by reference. If the receiver throws, `clientInvoke` unwind before reaching its wait. Those locals are destroyed, but the request remains in the event loop task set. Its response handlers can subsequently handle them

commit ACK

#### [build: require Cap'n Proto 1.0](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/527a099743e4a42391850165a994e6c3aa9347f5)

**commit message**
- [x] the allowCancellation annotation used by the cancellation tests does not exist in older versions
- [x] remove the configure time checks that only covered them
- [x] move the old deps CI config to 1.0.0

_thoughts_
fairly straightforward commit this is just bumping the capnp version to the newer one to support the allowCancellation annotation

Commit ACK
#### [test: Add testing for request cancellation](https://github.com/bitcoin-core/libmultiprocess/pull/342/changes/d51e391976500489668fa06b3db5b69912348403)

**commit message**
- [x] one test cancels an inflight `ProxyClient` call from another thread
- [x] another drops the response promise mid execution imitating non multiprocess clients

_thoughts_
the existing tests are good and exercise proper constrained environemnts
