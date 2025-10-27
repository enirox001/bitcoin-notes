### Review: wallet: Add `exportwatchonlywallet` RPC to export a watchonly version of a wallet

**PR:** [#33067](https://github.com/bitcoin/bitcoin/pull/32489)
**Author**: achow
**Type**: feature update

## Overview

This PR introduces a new RPC command `exportwatchonlywallet` that creates a watch-only version of an existing wallet by copying public descriptors, labels, transactions, and wallet flags while excluding private keys. This addresses a common pain point in airgapped Bitcoin setups.




- Does the implementation handle all descriptor types (P2PKH, P2WPKH, P2TR, multisig)?
- Are there sufficient unit tests covering error conditions?
- Is the RPC interface consistent with existing wallet commands?
- How does it handle wallet encryption/decryption during export?