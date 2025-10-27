# Review: test: refactor mempool_accept_wtxid

**PR:** [#33067](https://github.com/bitcoin/bitcoin/pull/33067)  
**Author:** naiyoma  
**Type:** Test refactor

## Overview

This PR modernizes the `mempool_accept_wtxid` test by replacing manual block generation with pre-mined chains, switching to MiniWallet for simplified transaction handling, converting a class to a function, and eliminating unnecessary variable assignments. The result is cleaner, faster, and more maintainable test code.

## Code Changes Analysis

### 1. Pre-generated Chain Usage

```diff
def set_test_params(self):
    self.num_nodes = 1
-   self.setup_clean_chain = True
+   self.setup_clean_chain = False

def run_test(self):
    node = self.nodes[0]
-   self.log.info('Start with empty mempool and 101 blocks')
-   # The last 100 coinbase transactions are premature
-   blockhash = self.generate(node, 101)[0]
+   self.log.info('Start with pre-generated blocks')
+   blockhash = self.nodes[0].getblockhash(1)
```

**Analysis:** This change eliminates the need to generate 101 blocks during test setup. By using `setup_clean_chain = False`, the test leverages Bitcoin Core's pre-mined regtest chain, significantly reducing initialization time. This is a common optimization pattern in Bitcoin Core tests.

### 2. MiniWallet Integration and Class-to-Function Refactor

The PR replaces the `ValidWitnessMalleatedTx` class with a `valid_witness_malleate_tx()` function and integrates MiniWallet:

```diff
-txgen = ValidWitnessMalleatedTx()
-parent = txgen.build_parent_tx(txid, 9.99998 * COIN)
-
-privkeys = [node.get_deterministic_priv_key().key]
-raw_parent = node.signrawtransactionwithkey(hexstring=parent.serialize().hex(), privkeys=privkeys)['hex']
-signed_parent_txid = node.sendrawtransaction(hexstring=raw_parent, maxfeerate=0)
+child_one, child_two = valid_witness_malleate_tx(wallet, node)
```

**Key improvements:**

- **MiniWallet handles transaction signing:** Eliminates manual key management and RPC calls for transaction signing
- **Simplified API:** The function returns ready-to-use child transactions instead of requiring manual parent transaction handling
- **Reduced complexity:** Class state management is unnecessary for this single-use case

### 3. Direct Property Access

```diff
-child_one_wtxid = child_one.wtxid_hex
-child_one_txid = child_one.txid_hex
-child_two_wtxid = child_two.wtxid_hex
-child_two_txid = child_two.txid_hex

-assert_equal(child_one_txid, child_two_txid)
-assert_not_equal(child_one_wtxid, child_two_wtxid)
+assert_equal(child_one.txid_hex, child_two.txid_hex)
+assert_not_equal(child_one.wtxid_hex, child_two.wtxid_hex)
```

**Analysis:** Eliminates unnecessary intermediate variables that were only used 2-3 times each. This follows the principle of avoiding premature optimization through variable caching when the performance benefit is negligible and readability suffers.

## Performance Impact

**Benchmark Results:**

- **Before:** ~36 seconds
- **After:** ~7-10 seconds
- **Improvement:** ~70-80% faster execution

The performance gain primarily comes from using pre-generated blocks instead of creating 101 new blocks during test initialization.

## Code Quality Assessment

**Strengths:**

- Maintains identical test coverage and assertions
- Follows Bitcoin Core testing best practices (MiniWallet usage, pre-mined chains)
- Reduces code complexity without sacrificing functionality
- Significant performance improvement

**Minor observations:**

- The transition from class to function is appropriate given the single-use nature
- MiniWallet integration aligns with ongoing Bitcoin Core test modernization efforts
- Direct property access improves code readability

## Verdict

**ACK** - This is a well-executed test refactor that improves performance, reduces complexity, and maintains test coverage. All CI checks pass, confirming the changes are self-contained and don't break existing functionality.

The PR successfully achieves its stated goals of modernizing the test while preserving its core purpose of testing witness transaction ID handling in the mempool.