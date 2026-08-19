_my thoughts are in italics_
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

@@ -124,7 +124,6 @@

void Connection::disconnect() // the event loop thread, like the destructor. assert(std::this_thread::get_id() == m_loop->m_thread_id); if (m_disconnected) return;

- m_disconnected = true;

// Cancel pending onDisconnect handlers first. Severing the connection // below completes m_network.onDisconnect() promises, and the registered

@@ -253,6 +252,8 @@

void Connection::disconnect() // stream. m_network.reset(); m_stream = nullptr;

+

+ m_disconnected = true;

} void Connection::waitDrained()
```

or a better or a better solution that make sure the the cleanup happens even if `shutdownWrite` fails

fourthly before this PR when a remote side disconnected, libmultiprocess had callbacks that would eventually remove the `Connection`
it does not necessarily call eht remove operation immediately, it can schedule into another task set. 

the new disconnect() wants different behaviour. such that it will disconnect and then call waitDrained later on. So it tries to reset the m_on_disconnect allbacks. 

but what if the callback has already progress one step further before reset happens

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