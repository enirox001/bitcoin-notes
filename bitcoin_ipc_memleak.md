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

lucasbaliero
- started a fresh session with only the sv2 apps bitcoin core and the essential system processes
	- maybe something might be eating the memory so we need a clean environment to properly test this
- server specs
	- 2 vcpu with 2gb of ram and ubuntu
- at baseline startup the full stack was running and the total memory usage was around 800mb
- after letting it run overnight the total ram usage was around 1.5gb
	- bitcoin core increased roughly by 160mb
	- with psrecord the apps were also growing as well
- over the next 8 hours the ram usage climbed from 1.5gb to 1.6gb
- since there was a bit more headroom, it was stressed a little
- alongside the already running apps 3 more pool instances were launched
- within 3 hours 
	- bitcoin core ram grew
	- total ram grew to 1.75gb
- **then shut down the extra pool instances**
	- bitcoin core did not release the memory 
	- the next day it was still around 1.69-1.75
	- seems bitcoincore does not reduce its memory footprint after clients disocnnect
- the next day
	- the vps was close to run out of ram
	- spun up an extra pool instance and the usage jumped to 1.82gb
	- closed the pool instance and the memory still didnt drop
	- a few hours later the OS killed bitcoin core

sjors
- asked if bitcoin core behaves better if compiled with the 30.x branch from source

lucasbaliero
- core feels more tolerant, the baseline ram is 100mb lower
	- could spin up more apps without core hitting big spikes
	- seems to be using less memory per connected app
- however after disconnection of the extra pool used for stress testing
	- the ram does not drop back down to the original baseline and it stays elevated

rusell
- it seems the that the memory usage increasing by 160mb overnight and 100mb over the next 6 hours
	- seems consistent with block templates not being freed, there might be two unknowns here
- we do not know wther the client is holding onto the block template references
	- if it is, the memory usage going up would be expected and it would be a client bug
- we do not know if the server is failing to release block templates when the client disconnects
- rss number does not decrease when the clients disconnect suggests that the server is failing to release memory
	- but it is not a reliable indicator as 
		- it only shows how much memory has been alocated by the os to the process
		- does not show how much memory is used in the process
		- when c library frees a memory
			- its normal for it to hold onto the pages that were in use and not return them to the os
- using a tool like heaptrack would be useful
- a good next step could be to add log statements to the `BlockTemplateImpl` constructor, destructor
	- write a python test program creating a template
	- see if the destructor is actually called 
		- on sudden disconnect 
		- when calling the destroy method
		- when deleting the reference
- could indicate if there is an obvious problem with the ipc server
	- there is a leak, but not sure if it is on the client or on the server

sjors
- the node message should contain a `destroy` entry everytime the client releases a template
- probably also a good idea to have the client emit a log message when it discard a template
	- if the client waits until disconnect will see a flood of `destroy` methods at this point
- if this is not happening could indicate a problem where core does not release the memory after the client disconnects
	- his could be done by adding logging to the constructor and destructor of the BlockTemplate

plebhash
- did some tests to address the logging suggestions
- using core v30 on mainnet and the sri pool with extra verbosity
- on the pool logs, after a certian block, destroyed 84 stale templates
- this come from the execution of `TemplateData::destroy_ipc_client` where we allocate one dedicated server thread
	- and then destroy each template sequentially
- on the core logs, we see 84 stale templates being destroyed

	**russell (response)**
	- wonders why the rust client is keeping 84 block template references
	- looking at the rust code it is currently designed to thold onto all templates
		- then free them when the tip changes
		- seems reasonable and should not use too much memory since they contain many of the same tx
	- logs also indicate that rust code is able to trigger block template deletions on the server side
	- logs show the rust client behaving well and deleting the templates it does not need?

		**plebhash response**
			- we conservatively keep all templates in memory up until the a chain tip update
			- as we could potentially receive a solution to any of them
			- transactions are not kept on the client side, it is fetched via the getblock method

russell
- following up on his comment added log statements to the BlockTemplateImpl constructor and destructor
- wrote a python test creating a block template object and calling the destroy method onit
- as soon as this is done
	- could see the destructor being called when this is done
	- and if not done we see it done regardless
- would indicate the leak is happening on the rust side
- and when the bug happens, the client is probably not
	- dropping references to allthe old templates or calling `destroy` on them
- doing either of those things should be sufficient to free memory on the server
	- might be a more subtle bug on the server side

russell
- would help to have some steps to reproduce

plebhash
- mentions how to possibly reproduce and measure it

russell
- had been experimenting with v0.1.0 pool_sv2 app on regtest
	- with a python script 
		- to generate transactions
		- connect to blocks quickly to make a memory leak happen without needing to wait a long time
- behaviour has been as expected
	- generating a lot of transactions without connecting nre blocks causes
		- block template to accumulate
		- memory usage to go up
	- but when a new block connected, the templates get released by the mining client
- one thing that surprisedd him was
	- if the client connected blocks too quickly
		- the mining client seemed to fall behind and
		- only released block templates after a delay
	- like the mining client would be calling destroy on the template 5000
		- when the block height was 5015 and was being connected
		- pausing the script would cause the mining client to catch up
- might need to try letting the code run a longer time to see if the memory usage can accumulate
	- a slow and gradual leak would be pretty difficult to debug and fix
	- perhaps a way to trigger the leak more quickly

sjors
- there is a hardcoded 10 second grace period where 
	- the template provider holds onto templates after the new tip connected
	- there for
		- try trying to relay the block anyways
		- giving pools the chance to verify the template and award shares

#### current research paths

preliminary: get a bitcoin core mainnet node synced and pruned up to 100gb of storage - might have to work around this
- [ ] simulate a node with high mempool activity (unsure what this looks like because i dont knwo what high mempool activity looks like)
- [ ] test the bitcoin core against different clients and measure the memory change (determine if there is one which doesnt or does not accumulate as much)
	- [ ] check for places spots where blocks are found like sjors mentioned
	- [ ] use the template memory management to do the same thing to see the different
	- [ ] use heaptrack to measure the memory usage on both bitcoin core and the apps to see where the memory is being allocated and not destroyed
- [ ] check  the server templates that are created for every client call
	- [ ] manually try to destroy them (or at least control to an extent how they are dropped)
	- [ ] check that they are destroyed deterministically 
- [ ] check if threads used in async using the context parameter behaves as it should
	- [ ] using the points as seen above do the same approach
		- [ ] manually try to destroy them (or at least control)
		- [ ] check that they are destroyed deterministically
- [ ] build a lean server that serves the templates, the point is not to use bitcoin core and perhaps not even libmultiprocess
- [ ] build a lean client that connects to the bitcoin core server to see if we have an issue
	- [ ] the point is to try and isolate the environment where the problem might be originating from

*a bit slower approach to solving the problem, and it could take longer time to debug and potentially find the solution, but the point is to find the place where the problem originates, and then perhaps use that knowledge to trigger a faster reproduction of the said problem*

