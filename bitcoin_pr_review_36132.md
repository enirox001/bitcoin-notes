_my thoughts are in italic, when i can put them_

### issue
the mining interface returns one nullable transaction for every requested txid or wtxid so callers can match results by position. 
When its `NodeContext` has no mempool, but there is a condition for it which violates the contract. both lookup methods return an empty vector for every non empty request and discard that positional mapping

### pr 36132
**[mining: preserve lookup result count without a mempool](https://github.com/bitcoin/bitcoin/pull/36132)**

#### desc
fix return a vector sized to the request when no mempool is available, default initialized null entries match the existing representation for identifiers absent from an available mempool. so callers recieve one response position per request in either state

_thoughts_
this fixes an edge case in the mining interface, when a caller requests txs by their id, the interface promises one per requested id, with a nullptr meaning that the transaction is unavailable

when the node has no mempool the pool of unconfirmed tx returned an empty list breaking that promise.

this is a defensive fix because in real life. the node that is started creates its own mempool on launch, so it is less than probable for this to happen irl

in the mining interface it is possible for the interface to be created before the mempool is created, using the `wait_loaded=false` so it is possible, but realistically

external ipc clients wait for chainstate loading by which point the mempool exists, and during shutdown the clients are disconnected before the mempool is destroyed.

i suppose there could be a situation where an external caller does something a bit odd and then makes it such that the node bypasses mempool creation, in that case then this could be a problem, so it is nice to have the defense

#### comments so far
jeanpablo said
- the case that is added builds an empty `NodeContext` which is nice, but the MiningTstingSetup still gets around it with nothing using it
	- l0rinc agreed and switched to use the BasicTestingSetup

sjors said
- an alternative would be to have it throw if there is no mempool, this would make the clients to not be misled that there are transactions even though we only have an empty list. However there is also a short time where the node has a mempool, and it is still being populated from the disk, in that case, it should throw too. But is fine with the approach here

looking at this i wonder if there is little benefits of having this

asked this question

"Concept ACK

I would understand the rationale behind this change, having the node return a list of nullptrs is better than having it return nothing, especially if the documentation already promises to at least return an entry per requested txid or wtxid

For my understanding, is there a current production path where these lookups run without the mempool, given that IPC `makeMining()` waits for the chainstate to load? Or is this primarily keeping the interface contract consistent for internal callers and tests?"

waiting for response would proceed once that is given