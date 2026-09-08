_my thoughts are in italics, when i can put them_

### issue
https://github.com/bitcoin/bitcoin/issues/33940

_we will go through the conversations, from each person to the end of the conversation to see the thought process and see the options that were not considered_

plebhash
- posted in [#33899](https://github.com/bitcoin/bitcoin/issues/33899#issuecomment-3568788623) that oom crashes were being experienced in https://github.com/stratum-mining/sv2-apps/pull/59#issuecomment-3568252007 
- initially suspected that it could be a thread related issue similar to the one reported in #33923 (this relates to a thread having issues when multiple requests are sent to the same thread) but it turns out it was the vps running out of its available ram
	- only happened after very long running sessions (12+ hours)
- leveraged psrecord to observe how bitcoin core from the 30.x branch was consuming ram across time 
	- **connected to mainnet for high mempool activity**
	- **for about 1 hour plus to have enough chain tip updates**
- ram consumption of the bitcoin core with sri pool connected to it
	- showed a clear upward trend in the ram consumption
- ran alonside sv2-tp 
	- showed the same upward trend that never gets throttled down
- suspects the root cause of this is related to template memory management

sjors
- plans to make a similar plot
	- without cpu
	- ideally with marks to show where blocks were found
-  if measuring the process memory instead of the tempate memory which a pr enables
	- will want to hold the memory itself constant
		- by picking some value for the maxmempool
		- waiting for it to fill before starting the measurement
		- also set dbcache to the minimum because that is also accruing
sjors
- initialiy thinks it is because a method does not have a context param
	- the destroy method is not invoked until sv2-tp disconnects
	- the node keeps holding on to templates even though the tp already pruned them

		**russell (response)**
		- imagines there is a bug causing the block templates not to be freed until the disconnect happens
		- does not think that the `context` would affet this
			- this only allows the method to run on an async thred without blocking the event loop instead of running on the event loop
			- but the thread the block was created on should not affect how it was destroyed 

plebhash
- do we have to explicitly call `destroy` to drop the references from memory on the client side?
- from understanding of capnp should be sufficient to drop it from memory
	- but on the other hand there must be a reason for `destroy` to exist

		**russell (response)**
		- just dropping the reference on the client side might not delete the server side right away
		- the reason for having an explicit destroy method 
			- to give clients a way to destroy the server side object
			- actually wait for it to be destroyed
		- if clients just drop their references to server side objects
			- the server side objects will be destroyed but it will happen asynchronously
		- its good practive to call `destroy` methods on objects which have them
			- to guarantee the objects are destroyed right away instead of asynchronously
			- this is most important if the objects have non trivial destructors
			- or if it matters what order objects get destroyed
		- block templates using a lot of memory in a memory constrained environment
			- could be another reason to call destroy explicitly
		- if adding `destroy` calls to the client fixes the memory leak (this is a bug)
			- memory should be freed as soon as reference are dropped even without explicit destroy calls

sjors
- libmultiprocess does this automatically, but only the sv2-tp uses that library
	- probably depends on the rust capnp library implemented
	- might be worth testing how the library behaves out of the box
	- with and without the fix
	- look for the `IPC server destroy` messages on the bitcoin core side

sjors 
- mentions destroy was not called by sv2-tp not because of the mssing context, but because of an unrelated regression
- memory leak is possibly because of not calling destroy
	
	**russell (response)**
	- the bheaviour has changed recently
		- before the 30.0
			- the commit mentions that the implemented ProxyServer objects is freed shortly after the client is freed
			- instead of being delayed until the connection is lost
		- before that change
			- libmultiprocess would only delete the server objects if the client explicitly called a `destroy` method
			- or when the connection was cleaned up
		- currently
			- it should delete them immediately 
			- even if a `destroy` method is never called
		- the previous behaviour didnt matter very much for libmultiprocess c++ client calls 
			- which destroy internally in their own destrcutors
			- but could matter for rust and python clients which dont do that

plebhash
- ran two sessions with [33936](https://github.com/bitcoin/bitcoin/pull/33936) where the client was the sv2 rust code
- both of them had at least one tip chain update (_i think chain updates happen after a block is found?_)
	- which should trigger templated being flushes from memory (would expect a sharp decline in memory consumption)
- first run does not call destroy, and it had the same steady increase in RAM
- in the second run, we call destroy after chain tip updates
	- there was an increase in at first, and then a sharp decline in memeory consumption
		- after a while it does start increasing though 

#### current research paths
- [ ] test the bitcoin core against different clients and measure the memory change (determine if there is one which doesnt or does not accumulate as much)
	- [ ] check for places spots where blocks are found like sjors mentioned
	- [ ] use the template memory management to do the same thing to see the different
- [ ] check  the server templates that are created for every client call
	- [ ] manually try to destroy them (or at least control to an extent how they are dropped)
	- [ ] check that they are destroyed deterministically 
- [ ] check if threads used in async using the context parameter behaves as it should
	- [ ] using the points as seen above do the same approach
		- [ ] manually try to destroy them (or at least control)
		- [ ] check that they are destroyed deterministically