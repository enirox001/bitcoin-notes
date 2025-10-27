PR: [#33477](https://github.com/bitcoin/bitcoin/pull/33477) 
Author: fjahr
Type: Refactor

## PR Analysis

This PR aims to implement `dumptxoutset` with rollback feature that was discussed a few times. It does not rely on `invalidateblock` and `reconsiderblock`. It instead creates a temporary copy of the coins DB, modifies this copy by rolling back as many blocks as necessary and then creating the dump from this temp copy DB.

---

## Files Changed 

1. `blockchain.cpp`: This is where the main refactor takes place. The code for creating rollbacks is refactored to move away from the `invalidateblock` and `reconsiderblock` approach creating the method for creating rollback form utxo snapshot. Also modifies rpc comments on the usability of the `dumptxoutset` rpc call.
2. `rcp_dumptxoutset`: modifies this test to simulate dumping of a utxo set at a specific height. It is simply to verify the changes to ensure that it works as expected

---

## Concept Review

Solid approach to the problem at hand, the problem showed itself at various points, one of which is the bug where at a certain height, dumping is just not possible. This solves it in a cleaner way.

Cleaner in the sense that the rollback mechanism is not hacky by putting two rpcs together and working with it as such. It would suffice and work as shown here in a PR i opened [33444](https://github.com/bitcoin/bitcoin/pull/33444). 

The approach taken by me takes the method of invalidating the block and then reconsidering it, but this solves it at a lower level by creating an internal temporary copy of the coins DB and the creating a dump from this temp copy.

It has it's advantages such as 
1. Forks cannot interfere with the rollback, effectively solving the bug mentioned here [32817](https://github.com/bitcoin/bitcoin/issues/32817
2. Network activity does not have to be suspended

Comes with slight disadvantages as well
1. Additional disk space is required for the copied coins DB
2. Performance is slower than other alternatives.

The pros outweigh the cons in this solution and hence can be justified.

Overall this is a strong **Concept ACK**. Addresses a bunch of problems while keeping it self contained.

---

## Code Review

Here is a commit-by-commit review:

* [rpc: Don't invalidate blocks on dumptxoutset](https://github.com/bitcoin/bitcoin/pull/33477/commits/f046286c0c5e1fef8b61da548cb97506f7a8b047): This commit consists of the main refactor, it begins by creating a new UniValue object out of the way.
  
  This consists of the NodeContext, ChainState, the CBlockIndex, fs::path for both the path and tmppath.
  
  The comments regarding the disabling of the network to facilitate rollback is then removed. Rollback no longer needs to disable the network and because of that the NetworkDisable class in the `blockchain.cpp` file.
  
  The TemporaryRollback class is also removed since it is no longer applicable, the new approach does not use the `InvalidateBlock` and the `ReconsiderBlock` present in this class as a result, it can be removed
  
  The `dumptxoutset` rpc method definition is then updated.
  
  The updates start with an update to the comments in the class, it removes the references to the network in the comments and clarifies the way this performs
  
  In the same method, the variable definitions for the `invalidate_index`, `disable_network` and `temporary_rollback` since the class definition has been removed, the variable definitions are subsequently not needed
  
  The code proceeds to define the result Univalue object, the chainstate is set to the current active chainstate
  
  It checks if the target_index is the tip, if this is the case, there is no need to rollback and proceeds to `CreateUTXOSnapshot` (The method definition for this is out of scope for this PR)
  
  if the target index is not the top, it proceeds to CreateRollBackUTXOSnapshot passing the required parameters (the definitions and comments related to `PrepareUTXOSnapshot` and `WriteUTXOSnapshot` in this case)
  
  Then comes the actual definition for the new `TemporaryUTXODatabase` class. It takes a path in it's constructor and uses the filesystem (fs) to create_directories. The deconstructor for this removes all the directories or throws a log info if it fails to asking for manual removal
  
  In the definition for the `CreateRolledBackUTXOSnapshot`, it starts by creating a temporary leveldb to store the UTXO set that is being rolled back, the path is built and then passed to the `TemporaryUTXODatabase` class defined above to create the temp_db_cleaner object
  
  Then the `DBParams` object is used to create a temporary database, the path defined above and other parameters are passed here to ensure that a temporary database is created. This is then used to create a unique pointer to a `CCoinsViewDB` object called `temp_db`
  
  The first main part of this method is then started, this consists of the copying the current UTXO set into a temporary database, to do this, the chainstate is flushed and then a `CCoinsViewCache` objects is created from the temp_db, the best block is then set for this `temp_cache`
  
  A unique pointer to the `CCoinsViewCursor` is then defined.
  *The CCoinsViewCursor is simply a cursor for iterating over CoinsView state*
  This is locked as well to prevent race conditions.
  
  `coins_count` is defined to 0, and then a while loop is initialized checking while the cursor is Valid (*TODO: Check what it means for cursor to be valid*). And then sets an `rpc_interruption_point `. A key and coin variable is defined (An Outpoint and Coin object respectively), and then the loop basically adds the coins to the temp_cache (There are underlying details here, but unnecessary for the point of discussion, only relevant for code review), and goes to the next coin in the view state in each iteration. Once that is complete, the `temp_cache` is flushed.
  
  Then comes the main rollback logic, It starts by defining the `block_index`, `rollback_cache`, `block_processed` and `res` variables. Also sets the best block of the `rollback_cache`.
  
  It then enters a while loop that checks if the block_index height is greater than the target height. If it is, an `rpc_interruption_point` is set, and it reads the block and disconnects the block (undo the effects of this block on the current UTXO set represented by coins signifying a rollback). It then increases the `block_processed` count, flushes the `rollback_cache` periodically and and goes back to the previous block index. 
  
  Once the while loop is done, it sets the best block of the `rollback_cache` to the hash of the target block index and flushes the cache
  
  Once the rollback is complete, it is logged, and then the stats is created (might fail to create). a cursor is created for the `temp_db` and the snapshot is written to the disk using `WriteUTXOSnapshot`.
  
  To round off this commit, the methods for checking the network in the `rpc_dumptxoutset.py` file is removed. Preparing for the changes in the next commit.
* [test: Add dumptxoutset fork test](https://github.com/bitcoin/bitcoin/pull/33477/commits/6d409d59704a026a56d18856ab2f9e90eea62186): This commit creates a method in the `rpc_dumptxoutset.py` file and performs the operation related to triggering a dumptxoutset. This is simply a validation function for the changes made, and it works fine, there does not seem to be any issues regarding the test.

The changes are focused around the rollback feature and the `dumptxoutset` method, not a lot of domain knowledge in regards to the coinsviewstate and the db cache, but the code does look good and the code is well written. So **Code ACK**

## Testing Review

the following tests were performed

1. Ran the project successfully
2. Ran the full functional test suite
3. Ran the specific rpc_dumptxoutset.py test

The tests pass as it should, no errors and no regressions related to the changes made.

**Tested ACK**

___

## Verdict

The change is well contained and good, it follows an internal approach to triggering rollbacks without using rpc hacks, which might work well, but are brittle, this is a welcome and better approach than the one i used when implementing this feature. 

**Solid ACK**