+++
title = "🅰️🅰️ Account Abstraction Beyond EIP-2938 🧵🎉"
date = 2020-10-13T06:53:00Z
+++
_Originally posted on [hackmd.io](https://hackmd.io/@SamWilsn/S1UQDOzBv)._

_Special thanks to [@adietrichs](https://twitter.com/adietrichs) and the rest of the Quilt team for review, content, and editing!_

## More Account Abstraction?

If you haven't yet, read [EIP-2938 Account Abstraction Explained][explainer] for some background on account abstraction (AA) and how [EIP-2938] implements it. To quickly summarize it here, EIP-2938 implements just enough AA to support single-tenant applications, with _minimal_ consensus changes and some new transaction propagation rules.

While the EIP lays the groundwork for AA and does provide a compelling solution for single-tenant use cases, like smart contract wallets, several use cases are not satisfied and require new features to be fully realized.

[explainer]: https://binarycake.ca/posts/account-abstraction-explained/
[EIP-2938]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md

<!--
## Use Cases

Several use cases benefit substantially from AA, and are summarized below along with some example implementations and the difficulties they face in current Ethereum.

### Smart Contract Wallet

Smart contract wallets bring the features of smart contracts to user accounts. Historically they are limited because only externally owned accounts (EOAs) can initiate transactions. [Gnosis Safe](https://gnosis-safe.io/) is a popular example.

### Mixer

Mixers, like [Tornado.cash](http://tornado.cash/), increase privacy by depositing funds of several people into a single pool and obfuscating the link between the depositor and the person withdrawing. The privacy of mixers in current Ethereum is somewhat compromised because withdrawing from a mixer requires funds to pay for gas, and those funds can be linked to a particular person. AA sidesteps the problem, and instead lets the mixer contract pay for gas itself.

### Arbitrage

Arbitrageurs compete to make trades to take advantage of price differences. Many of the transactions to execute these trades get reverted once the price differences are reconciled, wasting gas and exacerbating market inefficiency. [Uniswap](https://uniswap.org/)'s liquidity pools present an opportunity for arbitrage.

With AA, market contracts can define transaction validity rules that invalidate rejected trade transactions, instead of simply reverting them. The invalidated transactions cost no gas, and occupy no block space.

-->

## Features Beyond EIP-2938

Building upon the foundations of EIP-2938, there are several more features that can be implemented.

### Proxy Contracts with `DELEGATECALL`

#### What do we want?

Smart contract wallets often support upgradability by proxying calls to another contract. EIP-2938 alone does not support `DELEGATECALL` before `PAYGAS` because the target of the call may be destroyed, potentially invalidating a large number of pending transactions.

#### How do we get it?

Described in [EIP-2937], the `SET_INDESTRUCTIBLE` opcode conveniently prevents a contract from calling `SELFDESTRUCT`. EIP-2937 is a fairly minimal change, and is useful with or without AA.

To safely enable `DELEGATECALL` before `PAYGAS` from AA contracts, a node would check the first opcode of the library (callee) contract, and proceed if and only if it is `SET_INDESTRUCTIBLE`.

This lets AA contracts call out to libraries before `PAYGAS` to determine transaction validity. These libraries could be loaded from a dynamic address, enabling upgradability of the validation logic!

[EIP-2937]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2937.md

### Read-only Calls with `STATICCALL`

#### What do we want?

External read-only calls into an AA contract, like `eth_call`, are generally useful. Getting your account balance, the conversion rate between tokens, and simply reading your nonce are all incredibly common operations that use `STATICCALL`. The bytecode prefix in the EIP immediately stops all incoming calls, including static ones.

#### How do we get it?

Enabling `STATICCALL` is as simple as adding a new opcode `IS_STATIC` that returns true if the current frame of execution is read-only, and amending the AA bytecode prefix to allow external calls if `IS_STATIC` is true. Read-only calls cannot modify state, and therefore cannot invalidate other transactions.

### Read-write Calls

#### What do we want?

Read-only calls are great and all, but what about writing state into AA contracts from non-AA transactions? The most basic use case here is depositing ETH/tokens into a multi-tenant AA contract[^utxo], which requires updating a particular user's balance within the contract.

[^utxo]: Deposits are possible even without read-write calls, but it's a bit of a [kludge][bundle].

#### How do we get it?

The restrictions imposed by the EIP are designed to establish a ratio between how much an attacker has to spend in gas fees versus how many pending transactions are invalidated. The major problem with allowing read-write calls is that a single mined transaction could call into many AA contracts and invalidate their pending transactions, skewing this ratio. If the same ratio were to be preserved, read-write calls would be safe.

To this end, we introduce a new opcode `RESERVE_GAS`, taking one argument `N`, which guarantees a call consumes at least `N` gas. The exact specifics of how this opcode is implemented are still being specified (refund counter vs. a new counter.) Then we standardize an additional bytecode prefix for AA contracts which uses the new `RESERVE_GAS` opcode when called externally to enforce a minimum gas usage.

If calling into an AA contract is at least as expensive as targeting that contract directly, non-AA transactions can't invalidate any more transactions than directly targeting those contracts would have.

### Multiple Pending Transactions

#### What do we want?

Every multi-tenant[^1] use case (mixer, arbitrage, etc) requires that multiple transactions targeting the same AA contract propagate through the network simultaneously. Imagine having to send a transaction repeatedly to compete for the next nonce value, instead of just sending it and walking away.

EIP-2938 only propagates a single transaction at a time, and while [bundling transactions][bundle] does provide a partial workaround, the drawbacks make it impractical.

[^1]: Meaning in use by multiple external uncoordinated actors—human or not—at the same time.

[bundle]: https://hackmd.io/@SamWilsn/B1c0ZdaGP

#### How do we get it?

There are three problem areas that need to be solved before we can have multiple pending transactions per AA contract.

##### Replay Protection & Transaction Hash Uniqueness

Replay protection is the defense against another party taking your transaction and resending it after it has already been included in a block. Since the transaction fields don't change, without some external mechanism, the transaction remains valid. The nonce field in transactions serves that purpose today. As a secondary effect, the strictly-increasing nonce also guarantees that each transaction has a unique hash. As the signature covers all of the fields, including the nonce, another party cannot change the nonce without invalidating the transaction.

AA contracts, however, are free to choose what fields their signature scheme covers. They can, therefore, ignore the protocol nonce and define their own (or no) replay protection mechanism.

EIP-2938 handles nonces the same way as traditional transactions. Unfortunately using the nonce supplied when the transaction was created falls apart when there are pending transactions from multiple uncoordinated users. Once one of these transactions is mined, the contract's nonce increments and the rest of the pending transactions are dropped, even if the contract might still consider them valid.

To enable multi-tenant applications, whenever an AA contract's nonce changes, nodes "fix" the nonce of any pending transactions and revalidate them. For contracts with their own replay protection mechanism, this relegates the protocol nonce to the role of preserving hash uniqueness for transactions included in a block, effectively creating two distinct hashes[^2] for each transaction:
 * The *inclusion hash*, which is a new name for the traditional hash including the nonce. It uniquely refers to a particular transaction once mined, in a particular block, at a particular index within that block; and
 * The *nonceless hash*, which is a hash that refers to one or possibly more transactions, which may be propagating or included in a block.

Clients would likely have to provide an API endpoint for converting between the two hashes.

With malleable nonces comes additional validation effort every time a new block arrives. Pending transactions are revalidated after new blocks are validated, but this work isn't _part_ of block validation. Ideally all pending transactions can be revalidated before the next block arrives, and nodes can guarantee this by limiting how much cumulative validation gas each AA contract can consume. Quilt has [measured][dos] how different validation gas limits affect revalidation time, showing that even with generous gas limits, nodes should easily be able to complete revalidation in time.

[dos]: https://ethresear.ch/t/dos-vectors-in-account-abstraction-aa-or-validation-generalization-a-case-study-in-geth/7937

[EIP-2718]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2718.md

[^2]: Accepting proposals for better names!

##### Amplification Attacks

If an attacker spams non-AA transactions to the network, causing nodes to waste resources validating them, eventually a large portion of those transactions will make it into mined blocks, costing the attacker real money. Transactions with a low gas price get evicted from the pending pool, creating a price floor for this kind of attack.

For AA, a single mined transaction could invalidate many pending transactions, none of which make it into mined blocks, and therefore cost an attacker nothing. This is why the propagation rules in EIP-2938 are so restrictive.

The same simple solution above (a cumulative validation gas limit) mitigates amplification attacks as well. An attacker can still spam the network, but there is a clear bound on the ratio of validation work vs. paid transactions included in blocks. Quilt has done [extensive work][dos] indicating that, with reasonable bounds, nodes will remain able to cope with worst case validation load.

##### Crowding Out Transactions

Whether it's because of hardware constraints, or the cumulative gas limit mentioned above, nodes are only able to keep and propagate a limited number of transactions per AA contract. If nodes order pending AA transactions naively by gas price, a malicious user could crowd out other users by sending many high paying, mutually exclusive transactions targeting the AA contract. Only one of the attacker's transactions would make it on chain, reducing the contract's throughput at a low cost to the attacker.

Including an [EIP-2930]-style state access list and making it binding (state accesses outside the list fail) gives nodes enough information to effectively propagate multiple transactions. Transactions with disjoint access lists cannot invalidate each other, and are always safe to propagate. If two transactions' access lists intersect, nodes choose the one with a higher gas fee.

This approach is only sketched out in the EIP, and needs further research.

[EIP-2930]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2930.md

### Pure Validation Logic

#### What do we want?

Multiple pending transactions introduce the need to occasionally revalidate transactions. Some validation schemes (like zero-knowledge proofs) have a large one-time cost (to validate the proof) which cannot be invalidated by state or environment changes. If nodes are able to run these validation steps and cache the result, they can avoid the recurring cost during revalidation and safely enable higher validation gas limits.

#### How do we get it?

Building upon the `IS_STATIC` opcode and `STATICCALL`s into AA contracts from above, pure validation logic can be separated from regular validation and cached by standardizing a particular bytecode prefix, roughly implementing:

```
if (msg.sender == 0xffffffffffffffffffffffffffffffffffffffff) {
    let result = STATICCALL(this); // Or alternatively PURECALL
    return CALL(this, result);
} else if (msg.sender == address(this)) {
    if (IS_STATIC()) {             // Or alternatively IS_PURE
        let result = /* do one time validation */
        return result;
    }
} else {
    LOG1(msg.sender, msg.value);
    return;
}
```

If the AA contract reads from state or the environment during the initial `STATICCALL`, the node would disable caching and revert to the lower validation gas limit. Instead of handling these state reads as a special case, this behavior could be standardized with newly introduced opcodes `PURECALL` and `IS_PURE`.

In slightly more human-friendly terms, the contract first `STATICCALL`s itself to perform the pure validation step. Then the contract calls itself, supplying the result of the static call (which can be cached by the node), to do the state-dependent portion of the validation, invoke `PAYGAS`, and perform the rest of execution. Passing the cached value back to the contract during later revalidations could reuse the existing implementation of contract calls.

### Sponsored Transactions

#### What do we want?

Sponsored transactions, in the context of AA, are essentially setting `msg.sender` to a value other than the AA contract itself. Relayers could use sponsored transactions to act on behalf of the relayed account.

#### How do we get it?

[EIP-2711]'s specification of sponsored transactions is, unfortunately, incompatible with AA because it enshrines the signature format for sponsored transactions. As an alternative, we propose a new precompile that recovers a signature over the call data, and sets `msg.sender` appropriately.

[EIP-2711]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2711.md

## The End?

With only a handful of consensus changes and transaction pool rules, account abstraction goes from barely enough to support smart contract wallets to supporting exciting multi-tenant applications like mixers and exchanges.

Is account abstraction still not handling your use case? Doubt this is feasible? Come discuss with us in the [#account-abstraction][discord] channel over at the Eth R&D Discord!

[discord]: https://discord.gg/GU55yAW
