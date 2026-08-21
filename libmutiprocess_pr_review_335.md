_my thoughts are in italics, when i can put them_
### issue
assertion error was being raised whenever the ipc connection was being closed without waiting for the objects and threads associated with them to be freed. so if an ipc client makes a request and the node receives it and the node starts to shut down before the request is executed, asserts like this could be triggered.

### pr 335
**proxy-io.h: Add Connection `disconnect` and `waitDrained` method** [link](https://github.com/bitcoin-core/libmultiprocess/pull/335)

#### desc
add `Connection` class `disonnect` and `waitDrained` methods to provide flexibility when forcibly disconnecting from remote clients and servers
_this is a bit confusing to read would it be better if written as "add `disconnect` and `waitdrained` methods to the `Connection` class to provide more flexibility when forcibly disconnecting from remote clients and servers" ?_

without these methods only way to close ipc connections is to delete the `Connection` objects, this works but is not ideal because once the object is gone, it is difficult to track the state associated with the connection particularly
- Proxyserver objects that may still be alive because they are still executing async requests made before the disconnect, and since the `Connection` object has been destroyed and there is no way to wait for requests to finish exiting before disconnecting _there is a typo for exiting as existing in the desc, please update_ so individual ipc interfaces like the bitcoin mining interface would need to implement custom sync to avoid race conditions during shutdown. 
- ProxyClient objects that contain pointer to `Connection` objects. Currently `ProxyClient` object needs to register cleanup handlers with`Connection` objects to deal with `Connections` being delected, which complicates the shutdown logic. subsequent pr will drop this tracklist

_this is a good change, having a way to keep track of `Connection` state even after disconnect has happened is a nice improvement and a way to wait for async requests should resolve the issue here_

tests also run successfully.
**Concept ACK**

### commit by commit review
#### [39cc757 ipc: add Connection::disconnect() separating teardown from destruction](https://github.com/bitcoin-core/libmultiprocess/pull/335/changes/39cc757fba71871b154d915b02cf3967fe73fe0d)

**commit message**
- [x] split connection teardown out of the Connection destructor into an indempotent disconnect method 
- [x] the destructor will be delegating to this method now, this is a behaviour neutral refactor 
- [x] having a separate `disonnect method` allows severing a connection while keeping the Connection object alive, which the nect commits use to let shutdown wait for inflight server
- [x] disconnect cancels the `m_on_disconnect` handlers before severing the connection, these were implicitly canceled when the member was destroyed. When disconnect is called separately from destruction, this is required for correctness as severing the stream completes onDisconnect() and the registered handlers destroy the Connection object from the caller
- [x] disconnect explicitly releases `m_thread_pool` and `m_thread_map` so worker thread teardown happens at disconnect time wether the object is destroyed right away or not.

**Code Review**
in proxy-io.h we add the std::in_place to the m_network and make the m_rpc_system pass the piointer to the m_network, this is simply making the m_network to be able to be destroyed, previously we could not be able to destroy this, this inplace is making it an optional object here while somewhat preserving its behavior

the `disconnect()` method is added onto the `Connection` class with the comment stating how this method would behave when called
m_onDisconnect is now optioanl so disconnect() can destroy it to sever the transport while the object stays alive. Closing the stream is what makes the peer observe the disconnect. 
We also add a `m_disconnect` flag that is set once disconnect() has run. Only accessed in the event loop

in proxy.cpp the disconnect is added to run in the Connection destructor.

The method is then created, it checks if we are in the main loop thread, and then checks the flag. Cancels all pending onDisconnect handlers first, 

```
Lock lock{m_loop->m_mutex};

- while (!m_sync_cleanup_fns.empty()) {

- CleanupList fn;

- fn.splice(fn.begin(), m_sync_cleanup_fns, m_sync_cleanup_fns.begin());

- Unlock(lock, fn.front());

+ {

+ Lock lock{m_loop->m_mutex};

+ while (!m_sync_cleanup_fns.empty()) {

+ CleanupList fn;

+ fn.splice(fn.begin(), m_sync_cleanup_fns, m_sync_cleanup_fns.begin());

+ Unlock(lock, fn.front());

+ }

}
```
_why do we add this into another bracket? oh so it is a way to create a smaller lifetime scope for the mutex lock.lock is only alive within the brace so it is release immediately after the cleanup loop_  

and then in the same `disconnect()` method, release the thread capabilities owned by the connection, and then destroy the network and close the stream. closing the stream is what makes the peer observe the disconnect


*thoughts?* 

first of all, i think this commit message is a bit too verbose in simple terms the commit this, before we disconnect by `delete connection`

 after this commit we can call `connection.disconnect()` so that it shuts down the ipc connection without destroying the c++ connection object, and then later the actual `delete connection;` can be called to detroy the object. The destructor itself now just calls disconnect() and then destroys the `Connection` object later. The added detail is nice to have but i think it is a bit verbose.

secondly i think this is not exactly behavior neutral, the commit message itself says `Two details are new`, we now explicitly cancle `m_on_disconnect` handlers before severning the connection, and explicitly release `m_thread_pool` and `m_thread_map` during `disconnect()` rather than relying on member destruction.

The `m_on_disconnect` change is especially not something i would call behavior neutral as now we have to proactively cancel because `Connection` remains alive after the transport is severed and is no longer a consequence of destruction teardown. So even though the externally observable behaviour might seen unchanged, the lifetime and cancellation behaviour has changed, and i think that distinction matters

so the text saying

```
“This is a behavior-neutral refactor: the same steps run in the same order on destruction.”
```

is a bit misleading

thirdly `disconnect()` sets `m_disconnected = true`, later on we clean everything up, so i am unsure but if there was a scenario where one of the cleanups throws it would not complete the rest, this might not be a problem, but another call to disconnect() would be a no-op. 

i do not think all the operations after this can cause this to throw and lead to this, but `shutdownWrite()` might if it throws an exception other that the ones mentioned. 

a simple fix is to set the `m_disconnected = true` only after all teardown that must run has completed.

```diff --git a/src/mp/proxy.cpp b/src/mp/proxy.cpp
index 0aaa58a..8b9f458 100644
--- a/src/mp/proxy.cpp
+++ b/src/mp/proxy.cpp
@@ -124,7 +124,6 @@ void Connection::disconnect()
     // the event loop thread, like the destructor.
     assert(std::this_thread::get_id() == m_loop->m_thread_id);
     if (m_disconnected) return;
-    m_disconnected = true;

     // Cancel pending onDisconnect handlers first. Severing the connection
     // below completes m_network.onDisconnect() promises, and the registered
@@ -253,6 +252,8 @@ void Connection::disconnect()
     // stream.
     m_network.reset();
     m_stream = nullptr;
+
+    m_disconnected = true;
 }

 void Connection::waitDrained()
```

or a better or a better solution that make sure the the cleanup happens even if `shutdownWrite` fails

fourthly before this PR when a remote side disconnected, libmultiprocess had callbacks that would eventually remove the `Connection`
it does not necessarily call eht remove operation immediately, it can schedule into another task set. 

the new `disconnect()` wants different behaviour. such that it will disconnect and then call `waitDrained` later on. So it tries to reset the `m_on_disconnect` callbacks.

but what if the callback has already progress one step further before reset happens?

for example
```
rc closes -> onDisconnect callback fires -> 'delete Connectionn later' gets queued
```

then before that queued deletion runs
```
user calls disconnect() -> m_on_disconnect_resets()
```

this reset may be too late, because the deletion work is no longer owned by `m_on_disconnect`. Then
```
disconnect() returns (connection expected to still be alive) -> callback runs -> connection gets deleted
```

This violates the goal of this new system.

In PR 336 https://github.com/bitcoin-core/libmultiprocess/pull/336/changes/aa49a11c028f92aafa3867a59f9eb52cf9bdde6d this is made to use a weak_ptr, but i wonder if we should move it to those changes to this pr instead?

Or rather a small cancellation guard could be added to this PR such that it keeps the existing changes focused while preventing the potential regression.

A minimal change adding a weak cancelation token that have moved into the event loop queue.

```diff --git a/include/mp/proxy-io.h b/include/mp/proxy-io.h
index 1f77b26..d817eb6 100644
--- a/include/mp/proxy-io.h
+++ b/include/mp/proxy-io.h
@@ -576,8 +576,18 @@ public:
         // handler fires, do not call the function f right away, instead add it
         // to the EventLoop TaskSet to avoid "Promise callback destroyed itself"
         // error in the typical case where f deletes this Connection object.
+        const std::weak_ptr<void> guard{m_on_disconnect_guard};
         m_on_disconnect->add(m_network->onDisconnect().then(
-            [f = std::forward<F>(f), this]() mutable { m_loop->m_task_set->add(kj::evalLater(kj::mv(f))); }));
+            [f = std::forward<F>(f), guard, this]() mutable {
+                m_loop->m_task_set->add(kj::evalLater(
+                    [f = kj::mv(f), guard]() mutable {
+                        // The connection-owned TaskSet may have already handed
+                        // this callback to the event-loop TaskSet by the time
+                        // disconnect() cancels it. Only run it if the
+                        // connection has not been disconnected in between.
+                        if (guard.lock()) f();
+                    }));
+            }));
     }

     EventLoopRef m_loop;
@@ -587,6 +597,10 @@ public:
     //! disconnections, if the connection is closed locally first by deleting
     //! this Connection object.
     std::optional<kj::TaskSet> m_on_disconnect{std::in_place, m_error_handler};
+    //! Lifetime token checked by onDisconnect handlers after they are handed
+    //! off to the EventLoop TaskSet. Reset by disconnect() so a handler already
+    //! queued there cannot run after local teardown.
+    std::shared_ptr<void> m_on_disconnect_guard{std::make_shared<char>()};
     //! Wrapped in std::optional so disconnect() can destroy it (and m_stream
     //! below) to sever the transport while this object stays alive. Closing
     //! the stream is what makes the peer observe the disconnect: it reads EOF
diff --git a/src/mp/proxy.cpp b/src/mp/proxy.cpp
index 0aaa58a..06063f8 100644
--- a/src/mp/proxy.cpp
+++ b/src/mp/proxy.cpp
@@ -133,6 +133,7 @@ void Connection::disconnect()
     // harmful when disconnect() is called separately by code that keeps using
     // the object afterwards (e.g. code waiting for in-flight calls to finish
     // before destroying it).
+    m_on_disconnect_guard.reset();
     m_on_disconnect.reset();

     // Try to cancel any calls that may be executing.
```

this change closes the gap where resetitng `m_on_disconnect` was too late because the callback had already moved into the Evenlopp task set

fifth(ly?) the listener keeps a now counter of the active connections added in https://github.com/bitcoin-core/libmultiprocess/pull/269/changes/39a10ce8958ef9d49d0ec82acd3a6d506d5c8fee

when it is full it stopes accepting new connections, and when a client disocnnect, a callback decreases the counter and the listener can start accepting again.

But when the server calls `disconnect()` it cancels that callback. The connection closes, but the counter does not change, so the listener might think it is full and never accept another connection

Added this change to so that the listeener count can be updated for every disconnect, while automatic deletion happens only for remote disconnects

```diff --git a/include/mp/proxy-io.h b/include/mp/proxy-io.h

index 1f77b26..30627ec 100644

--- a/include/mp/proxy-io.h

+++ b/include/mp/proxy-io.h

@@ -1016,10 +1016,12 @@

void _Serve(EventLoop& loop, kj::Own<kj::AsyncIoStream>&& stream, InitImpl& init auto it = loop.m_incoming_connections.begin(); MP_LOG(loop, Log::Info) << "IPC server: socket connected."; if (loop.testing_hook_connected) loop.testing_hook_connected();

- it->onDisconnect([&loop, it, on_disconnect = std::forward<OnDisconnect>(on_disconnect)]() mutable {

+ it->addSyncCleanup([on_disconnect = std::forward<OnDisconnect>(on_disconnect)]() mutable {

+ on_disconnect();

+ });

+ it->onDisconnect([&loop, it]() mutable {

MP_LOG(loop, Log::Info) << "IPC server: socket disconnected."; loop.m_incoming_connections.erase(it);

- on_disconnect();

if (loop.testing_hook_disconnected) loop.testing_hook_disconnected(); }); }
```

This test could be added to validate this as well

```diff --git a/test/mp/test/listen_tests.cpp b/test/mp/test/listen_tests.cpp

index a9d4dca..240af3f 100644

--- a/test/mp/test/listen_tests.cpp

+++ b/test/mp/test/listen_tests.cpp

@@ -265,6 +265,29 @@

KJ_TEST("ListenConnections enforces a local connection limit") KJ_EXPECT(client3->client->add(3, 4) == 7); }

+KJ_TEST("ListenConnections resumes after a local disconnect")

+{

+ ListenSetup server(/*max_connections=*/1);

+

+ auto client1 = std::make_unique<ClientSetup>(server.listener.MakeConnectedSocket());

+ server.WaitForConnectedCount(1);

+ KJ_EXPECT(client1->client->add(1, 2) == 3);

+

+ auto client2 = std::make_unique<ClientSetup>(server.listener.MakeConnectedSocket());

+ (**server.m_loop_ref).sync([] {});

+ KJ_EXPECT(server.ConnectedCount() == 1);

+

+ EventLoop& loop{**server.m_loop_ref};

+ loop.sync([&] {

+ KJ_REQUIRE(loop.m_incoming_connections.size() == 1);

+ loop.m_incoming_connections.front().disconnect();

+ loop.m_incoming_connections.pop_front();

+ });

+

+ server.WaitForConnectedCount(2);

+ KJ_EXPECT(client2->client->add(2, 3) == 5);

+}

+

KJ_TEST("ListenConnections accepts multiple connections") { // With max-connections=2, two clients should be accepted and usable at the
```

sixth

Connection can now remain alive after it has been disconnected, but the class doe not define which methods are safe to call afterwards.

Previously disconnectiong meant destroying the whole object, but now the connection object still exists. This is needed so callers can use methods such as `waitDrained` But other methods still behave as if the connection is active.

An assertion or runtime check could be helpful here?

This commit does a lot of good things right, but there are changes that are still needed here in order to make this ready, the most important being the disconnect callback and the listener count (perhaps a bit biased about the 2nd one, it is my pr after all).

Commit (Concept) ACK

#### [ipc: add Connection::disconnect() separating teardown from destruction](https://github.com/bitcoin-core/libmultiprocess/pull/335/changes/631d8d9d439e22ea782d469c5ce4c75e2ee3d58c)

**commit message**
- [x] Add a per connection ServerObjectTracker counting live proxyServer objects
	- [x] incremented in the ProxyServerBase constructor and decremented in its destructor
	- [x] with waitDrained() blocking until the count reaches zero and pendingServerObjects() expose it for logging
- [x] Disconnecting a connection cancel s the KJ promise of an in flight call
	- [x] but a c++ server method body already dispatched to a worker thread runs to completion
	- [x] counting live server objects turns capnp object lifetime rules into a quiescene signal
		- [x] a proxy server object is not destroyed until its outstanding calls finish
			- [x] after discoonnect() the count drains to zero exactly ehrn no server call body is executing
		- [x] waiting for this lets shutodown code avoid freeing application state that a still running call body dereferences
- [x] The tracker is held via `shared_ptr` by the Connection
	- [x] by every ProxyServer object kept alive by inflight calls can outlive the Connection on some teardown paths
	- [x] their destructors must decrement state that is still valid
- [x] It must be declared before the rpc_system whose construction creates the bootstrap server object that registers itself with the tracker

this second commit adds a way to wait until server side ipc calls have actually finished after disconnection. The commit adds a counter for each connection, number of live server objects, so when the proxyserver is created it adds the counter and when it is destroyed it subtracts. A server object is kept alive while one of its calls is running, hence after disconnect

if the counter is more than 0 it means one or  more bodies may still be running, `waitDrained()` waits fo the counter to become 0.

It uses a `shared_ptr` so that the inflight call can keep its `ProxyServer` alive after the `Connection` object has been destroyed. It must still decrement after that, so it cannot be stored inside the Connection. That is why the Connection and ProxyServer share the drain counter

The code looks good to me *i do not feel like explaining the code here tbh*

*thoughts*

first thoughts commit message and documentation is doing too much and overexplaining, this looks and sounds lot like unflitered AI generated code noise, and i do not know how to feel about it. regardless would mention that.

secondly the waitDrained method mentions that it should only be called after `disconnect` has been called, but it does nothing to assert to make sure that it has, not completely sure if there is a code path that could trigger it without the disconnect running, but i think we can assert this by:

```diff --git a/src/mp/proxy.cpp b/src/mp/proxy.cpp
index 0aaa58a..6a318b0 100644
--- a/src/mp/proxy.cpp
+++ b/src/mp/proxy.cpp
@@ -261,6 +261,7 @@ void Connection::waitDrained()
     // bodies sync() back to the event loop to deliver their results, and
     // server objects are destroyed on the event loop thread.
     assert(std::this_thread::get_id() != m_loop->m_thread_id);
+    assert(m_disconnected);
     m_server_objects->wait();
 }
```

thirdly the name waitDrained could be more clear if it is called waitServerCalls drained? or at least document its exact scope a bit clearly

apart from that the commit looks generally good, nothing out of the blue for me at the moment

Commit ACK
#### [test: cover draining in-flight server call after disconnect](https://github.com/bitcoin-core/libmultiprocess/pull/335/changes/092d1db8fe7531e220c90bab0c4a78c83bdd978c)

**commit message**

add a deterministic mptest regression test
- [x] hold a server method body in flight on a worker thread
- [x] call disconnect()
- [x] assert that waitDrained() blocks until the body finishes
- [x] and its server object is destroyed
- [x] also covers destroying already disconnected connection

in this pr the test creates the situation where it starts an ipc server call on a worker thread, Make that call pause before it finishes, disconnect the connection, call waitdrained from another thread and then checks that waitDrained does not return while the server call is paused

*thoughts*

the test commti still has the same wordy problem i mentioned, but that is to be expected with llm output i suppse we can roughly half it without losing meaning

secondly
The 20 ms check is a bit too scheduler dependent, if the drain thread has not been scheduled during that interval, drained remains false even if `waitDrained()` is broken and would return immediately causing false pass.

We could add a hook here that runs only when `ServerObject::wait()` sees a non zero count and is about to wait

```diff --git a/include/mp/proxy-io.h b/include/mp/proxy-io.h
index 1f77b26..ab506b6 100644
--- a/include/mp/proxy-io.h
+++ b/include/mp/proxy-io.h
@@ -494,12 +494,14 @@ struct ServerObjectTracker
     void wait()
     {
         Lock lock(m_mutex);
+        if (m_count != 0 && testing_hook_wait) testing_hook_wait();
         m_cv.wait(lock.m_lock, [this]() MP_REQUIRES(m_mutex) { return m_count == 0; });
     }

     mutable Mutex m_mutex;
     std::condition_variable m_cv;
     size_t m_count MP_GUARDED_BY(m_mutex){0};
+    std::function<void()> testing_hook_wait;
 };

 //! Object holding network & rpc state associated with either an incoming server
diff --git a/test/mp/test/test.cpp b/test/mp/test/test.cpp
index 5bccb86..eec52f6 100644
--- a/test/mp/test/test.cpp
+++ b/test/mp/test/test.cpp
@@ -474,13 +474,17 @@ KJ_TEST("Waiting for in-flight server call to finish after disconnect")
     // A drain must block while the body runs and return only once it
     // finishes, which is what Ipc::disconnectIncoming relies on during
     // shutdown.
+    std::promise<void> drain_waiting;
+    connection->m_server_objects->testing_hook_wait = [&] { drain_waiting.set_value(); };
     std::atomic<bool> drained{false};
     std::thread drain_thread([&] {
         connection->waitDrained();
         drained = true;
     });

-    // The body is still blocked, so waitDrained() must not have returned.
+    // Wait until waitDrained() has observed the live server object and is
+    // about to block, then verify it does not return while the body is blocked.
+    drain_waiting.get_future().get();
     std::this_thread::sleep_for(std::chrono::milliseconds(20));
     KJ_EXPECT(!drained);
```

The test will then wait for that hook before starting the 20ms check. This ensures the drain thread has entered `wait` and observed pending work. This substantially reduces the possibility of a false positive

thirdly
I think using exact count of one is a bit brittle the test only needs to establish that something remains in flight, this would be less implementation specific

```diff --git a/test/mp/test/test.cpp b/test/mp/test/test.cpp
index 5bccb86..da47fde 100644
--- a/test/mp/test/test.cpp
+++ b/test/mp/test/test.cpp
@@ -463,13 +463,13 @@ KJ_TEST("Waiting for in-flight server call to finish after disconnect")

     // The FooInterface server object is the connection's only counted server
     // object, and its call body is executing.
-    KJ_EXPECT(connection->pendingServerObjects() == 1);
+    KJ_EXPECT(connection->pendingServerObjects() > 0);

     // Disconnect. This cancels the call's promise (the client above sees the
     // disconnect error), but the body is still blocked on the worker thread,
     // so its server object must still be alive.
     foo->m_context.loop->sync([&] { connection->disconnect(); });
-    KJ_EXPECT(connection->pendingServerObjects() == 1);
+    KJ_EXPECT(connection->pendingServerObjects() > 0);

     // A drain must block while the body runs and return only once it
     // finishes, which is what Ipc::disconnectIncoming relies on during
```

*the next thought i have is dependent on the author accepting the proposal i made to move the change from the other pr to this one or at least take the approach i took in order to make this pr at least okay to be used*

overall this is a test commit and does all it mentions it does quite well.

Commit ACK

#### [Add EventLoop::incoming_connections() that returns std::views::all of the m_incoming_connections list.](https://github.com/bitcoin-core/libmultiprocess/pull/335/changes/a40189f5bbed89f09e5d63ce8377dc90574983f6)

**commit message**
- [x] Add incoming_connections that returns std::views::all of the m_incoming_connections list
- [x] Currently the list holds Connection by value
- [x] When keepconn+track later changes the list to `list<shared_ptr<Connection>>`
- [x] the accessor will be updated to return a transform view
- [x] core code taht iterates via this accessor compiles unchanges across that type change

this commit adds a method `incoming_connections` on the `Eventloop` that provides a standard way for callers to iterate over incoming connections. This directly enables the subsequent pr 336 that changes the list of connections to a shared_ptr list, without this it would be accessing by value unless they made it a shared_ptr as well, but this accessor puts a small adapter in front of the internal list

*thoughts*

the commit message is not good at all, it is not clear in anyway, it is a bit obfuscated tbh, it is hard to understand. Had to use an llm to understand waht is going on here

Commit ACK

#### [Fix thread map teardown race causing use-after-free on disconnect](https://github.com/bitcoin-core/libmultiprocess/pull/335/changes/901a090da03ed6b04d46a4040ed16fdeb238e7fd)

**commit message**
_a bit wordy commit message, i wonder if this can be made a bit succint?_

- [x] fix a race condition between a thread exiting after making an ipc call and a connection being destroyed by its onDisconnect handler on the event loop thread _i should be able to write a test to verify this_
	- [x] the race was between `~ThreadContext` destroying the thread-local request_threads/callback_thread maps with no locking
	- [x] and the `SetThread` cleanup function (run by `Connection::disconnect`) erasing entries from those maps on the event loop thread
	- [x] When the two ran concurrently, both could destroy the same `ProxyClient<Thread>` object
	- [x] The SetThread cleanup reset `m_disconnect_cb` just before `~ProxyClient<Thread>` checked it unsynchronized
	- [x] so the exiting loop map erase destroyed it too
	- [x] The double destruction consumed `m_context.cleanup_fns` on one threadm so the other never unregistered the `Connection::disconnect` then invoked that callback on the thread freed map node
- [x] fix by making the entry removal the synchronization point deciding which side destroys each `ProxyClient<Thread>`
	- [x] Add an explicit `~ThreadContext` that removes map entries one at a time under `Waiter::m_mutext` and destroys each removed node rather than releasing the mutex instead of destroying the maps unlocked
	- [x] Change the `SetThread` cleanup function to look its entry up by connection key under the Waiter mutex instead of dereferencig the captured map iterator, extract it, and destroy the nde outside the lock, folowing the same pattern as `PassField` already uses for mp.Context arguments, if the entry is gone, the owning thread extracted if first and is responsible for destroying it
	- [x] guard the removesyncCleanup call in `~ProxyClient<Thread>` with a `m_context.connection` check, because when the entry was extracted by the ThreadContext destructor first, a concurrent disconnect still runs both the `SetThread` cleanup and the ProxyClientBase disconnect callback, leaving `m_disconnect_cb` set but pointing at a spliced out list iterator that must not be passed to removeSyncCleanup. The disconnect callback nulls `m_context.connection`, and posted functions cannot interleave with `Connection::disconnect` on the event loop thread, so a null connection reliably indicaes this case
The rase is long standing on master via connections created by `ConnectStrean`m whose onDisconnect handler delets the client connection on the eventloop thread when the peer disconnects while an existing thread may be running `~ThreadContext`

This commit is fixing a race condition where two threads could try to delete the same thread client object.
There are maps inside `ThreadContext` that  contain `ProxyServer<Thread>` objects associated with connections.

Two things can remove the entry
1. the client thread exits and destroys its thread context
2. the event loop thread disconnects a conncetion and removes the connection entries

previously these could happen simultaneously. The commit here makes the both sides use the same mutex when taking an etry out of the map. Whichever thread removes the entry first is responsible for destroying it, the other thread sees that it is already gone and does nothing.

*thoughts*
commit seems good, i did not look into this in depth, it is still a bit all over the place, but the code and the rationale behind the change looks good

one thing here is the `Waiter` documentation says
```
//! This mutex can be held at the same time as
//! EventLoop::m_mutex as long as Waiter::mutex is locked first and
//! EventLoop::m_mutex is locked second.
```

but the new commit says
```
//! Waiter::m_mutex must not be held when EventLoop::m_mutex is
//! acquired
```

It also says releasing the waiter mutex avoids locking the Waiter mutex before the Eventloop mutex, these rules cannot both be correct.

I think an actual order should be identified and updated here

Russell responded saying 

"Hmm, this is an interesting finding. But if there is a bug here, it seems like a pre-existing one, not something caused by this change or made worse by it.

You're saying if a remote disconnect happens first, and the m_network->onDisconnect() callback executes, but the kj::evalLater callback inside it does not execute yet, and if within that interval, the local process decided to delete the connection, then the connection could be deleted twice.

This does seem like it might be possible, and I'd want to look into it a little more and write a test. I'd still be inclined to save a fix for a different PR, and I believe as you pointed out #336 might fix this."[reply](https://github.com/bitcoin-core/libmultiprocess/pull/335#discussion_r3825422809)

_thoughts_

this is not the point i intended to pass across, it was more 
- remote callback queued
- local code calls disconnect(), intending to keep the connection alive
- local code retains the connection pointer for waitDrained
- queued callback erases and destroys the Connection
- shutdown code uses the dangling connection pointer

it is not a 
- local code deletes the connections
- queued callback deletes it again

But this led me to ask could it be possible for the actual point thought of by Russell be justified? Could it be possible for the connection object to be deleted twice, once by the object and another time by a queued callback?

so i added a ttest to verify this, first of all, i added a hook to be called before an onDIsconnect callback is queued on the eventloop

```diff --git a/include/mp/proxy-io.h b/include/mp/proxy-io.h
index 1f77b26..100bd10 100644
--- a/include/mp/proxy-io.h
+++ b/include/mp/proxy-io.h
@@ -378,6 +378,9 @@ public:

     //! Hook called on the event loop thread when a client has disconnected.
     std::function<void()> testing_hook_disconnected;
+
+    //! Hook called before an onDisconnect callback is queued on the event loop.
+    std::function<void()> testing_hook_before_on_disconnect_queued;
 };

 //! Single element task queue used to handle recursive capnp calls. (If the
@@ -577,7 +580,12 @@ public:
         // to the EventLoop TaskSet to avoid "Promise callback destroyed itself"
         // error in the typical case where f deletes this Connection object.
         m_on_disconnect->add(m_network->onDisconnect().then(
-            [f = std::forward<F>(f), this]() mutable { m_loop->m_task_set->add(kj::evalLater(kj::mv(f))); }));
+            [f = std::forward<F>(f), this]() mutable {
+                if (m_loop->testing_hook_before_on_disconnect_queued) {
+                    m_loop->testing_hook_before_on_disconnect_queued();
+                }
+                m_loop->m_task_set->add(kj::evalLater(kj::mv(f)));
+            }));
     }

     EventLoopRef m_loop;
```

and then wrote a test that cancels an already queued onDisconnect callback

```diff --git a/test/mp/test/test.cpp b/test/mp/test/test.cpp
index 5bccb86..4fd906c 100644
--- a/test/mp/test/test.cpp
+++ b/test/mp/test/test.cpp
@@ -291,6 +291,42 @@ KJ_TEST("Calling IPC method after server connection is closed")
     EXPECT_EXCEPTION(foo->add(1, 2), "IPC client method call interrupted by disconnect.");
 }

+KJ_TEST("Destroying a connection cancels an already queued onDisconnect callback")
+{
+    std::promise<bool> result;
+    std::thread loop_thread{[&] {
+        EventLoop loop("mptest", [](mp::LogMessage) {});
+        auto pipe = loop.m_io_context.provider->newTwoWayPipe();
+        auto server_connection =
+            std::make_unique<Connection>(loop, kj::mv(pipe.ends[0]), [&](Connection& connection) {
+                return capnp::Capability::Client(kj::heap<ProxyServer<messages::FooInterface>>(
+                    std::make_shared<FooImplementation>(), connection));
+            });
+        auto client_connection = std::make_unique<Connection>(loop, kj::mv(pipe.ends[1]));
+        auto client = client_connection->m_rpc_system->bootstrap(ServerVatId().vat_id).castAs<messages::FooInterface>();
+        bool callback_ran{false};
+
+        server_connection->onDisconnect([&] { callback_ran = true; });
+        loop.testing_hook_before_on_disconnect_queued = [&] {
+            loop.m_task_set->add(kj::evalLater([&] {
+                server_connection.reset();
+                loop.m_task_set->add(kj::evalLater([&] {
+                    client = nullptr;
+                    client_connection.reset();
+                    result.set_value(callback_ran);
+                }));
+            }));
+        };
+
+        loop.m_task_set->add(kj::evalLater([&] { client_connection->disconnect(); }));
+        loop.loop();
+    }};
+
+    const bool callback_ran{result.get_future().get()};
+    loop_thread.join();
+    KJ_EXPECT(!callback_ran);
+}
+
 KJ_TEST("Calling IPC method and disconnecting during the call")
 {
     TestSetup setup{/*client_owns_connection=*/false}
```

and sure as rain? the test fails

this shows that onDisconnect callback can execute after its owning `Connection` has already been destroyed