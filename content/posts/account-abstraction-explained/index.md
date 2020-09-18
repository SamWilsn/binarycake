+++
title = "🅰️🅰️ EIP-2938 Account Abstraction Explained 🧵👟"
date = 2020-09-17T23:40:00Z
+++
_Originally posted on [hackmd.io](https://hackmd.io/@SamWilsn/ryhxoGp4D)._

## What is Account Abstraction

As of the Muir Glacier hard-fork, out of Ethereum's two kinds of accounts—externally owned accounts (EOAs, like your MetaMask wallet) and smart contracts—only EOAs may pay gas fees for transactions. Lifting that restriction and allowing custom validity logic is, at an extremely high level, Account Abstraction (AA).

In this article, we want to give a brief and understandable explanation of [EIP-2938], our proposal to bring AA to Ethereum.

### A Concrete Example

The best way to explore AA is to see what you can build with it! So without further ado, here's a 2-of-2 multisig wallet that pays for its own transactions:

```solidity
// SPDX-License-Identifier: MPL-2.0

pragma solidity ^0.7.1;
pragma experimental ABIEncoderV2;      // Enables structs in the ABI.

account contract TwoOfTwo {            // Note the new `account` keyword!
                                       // This marks the contract as
                                       // accepting AA transactions, and
                                       // makes solidity emit a special
                                       // prelude. More on that later.

    struct Signature {
        uint8 v;
        bytes32 r;
        bytes32 s;
    }

    address public owner0;            // Making calls from this account
    address public owner1;            // requires two signatures, making
                                      // this a 2-of-2 multisig.
    
    constructor(
        address _owner0,
        address _owner1
    ) payable {
        owner0 = _owner0;
        owner1 = _owner1;
    }
    
    function transfer(                // Emulates a regular Ethereum
        uint256 gasPrice,             // transaction, but with a new
        uint256 gasLimit,             // validity requirement:
        address payable to,           //
        uint256 amount,               //
        bytes calldata payload,       //
        Signature calldata sig0,      // Two signatures instead of one!
        Signature calldata sig1
    ) external {
        bytes32 digest = keccak256(   // The signature validation logic
            abi.encodePacked(         // for AA contracts is implemented
                this,                 // in the contract itself. This
                gasPrice,             // gives contracts a ton of
                gasLimit,             // flexibility. You don't even need
                to,                   // to use ECDSA signatures at all!
                amount,
                tx.nonce,             // Newly exposed!
                payload
            )
        );

        address signer0 =             // If either signature is invalid
            recover(digest, sig0);    // the contract reverts the
        require(owner0 == signer0);   // transaction.
        
        address signer1 =             // Since the revert happens before
            recover(digest, sig1)     // `paygas` is called, the entire
        require(owner1 == signer1);   // transaction is invalid, and
                                      // this contract's balance is not
                                      // reduced.

        paygas(gasPrice, gasLimit);   // Signals that the transaction is
                                      // valid, and the gas price and
                                      // limit the contract is willing to
                                      // pay. Also creates a checkpoint:
                                      // changes before `paygas` are not
                                      // reverted if execution fails past
                                      // this point.
        
        (bool success,) =
            to.call{value: amount}(payload);
        require(success);
    }
    
    function recover(
        bytes32 digest,
        Signature calldata signature
    ) private pure returns (address) {
        return ecrecover(
            digest,
            signature.v,
            signature.r,
            signature.s
        );
    }
}
```

This is a rough sketch of how AA support could look in solidity, based on our early prototypes (available in the [playground]).

## What is EIP-2938?

[EIP-2938] is a specification for one flavor of AA, designed to be fairly simple to implement while allowing new and more powerful features to be developed in the future.

### Consensus Changes

The EIP contains three consensus critical changes:

* Add a new [EIP-2718] transaction type with fields `[nonce, target, data]`, where `target` is the AA contract's address. Note the omission of `to`, `gas_price`, `gas_limit`, and signature fields. Transactions of this type set `tx.origin` to `0xffff...ff`; a special address known as the AA entry point.
* Add a `NONCE` opcode (`tx.nonce` in solidity) that pushes the transaction's nonce field.
* Add a `PAYGAS` opcode which:
    * Creates an irrevertible checkpoint, guaranteeing state changes before `PAYGAS` cannot be undone by subsequent code.
    * Accepts a version argument, followed by a variable number of arguments. Initially the version is always `1`, but exists for future compatibility with, for example, [EIP-1559].
    * In this first version:
        * Accepts a gas price and gas limit, signaling the contract's willingness to pay for the transaction.

[EIP-1559]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md
[EIP-2718]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2718.md

### Other Changes

When the existing transaction pool logic is combined with AA's arbitrary transaction validity a new[^1] type of attack on the Ethereum network is possible: a single transaction included in a mined block can invalidate a large number of previously valid pending transactions. Under a sustained attack, nodes would waste significant computation validating, propagating, then discarding these transactions. The EIP introduces a number of transaction pool restrictions to mitigate this attack, bringing the risk to a level comparable to non-AA transactions.

First, nodes will not accept AA transactions with nonces higher than the currently valid nonce. If multiple transactions arrive with the same nonce, only the highest paying transaction will be kept. When the currently valid nonce for an AA contract changes (i.e. upon receipt of a new block with a transaction for that contract), nodes will drop the pending transaction for that contract, if one exists.

Second, the EIP proposes a standard bytecode prefix for AA contracts. For non-AA invocations (i.e. `msg.sender != 0xffff...ff`) the prefix emits a log of `msg.sender` and `msg.value`. For AA invocations, the prefix passes execution to the main contract. Nodes will drop any AA transactions targeting a contract which doesn't begin with this standard prefix. Over time, more prefixes can be added (without a hard fork) to allow further functionality.

Finally, encountering any of the following conditions before `PAYGAS` will cause the node to immediately drop the transaction:
 * An environment opcode (`BLOCKHASH`, `COINBASE`, ...);
 * Retrieving the `BALANCE` of any account, including the target itself;
 * An external call/create (except to precompiles);
 * An external state access that reads code (`EXTCODEHASH`, ...);
 * More than a fixed validation gas limit (currently 90,000) has been consumed.

These restrictions ensure that the only state accessible to the validity logic is state internal to the AA contract, and that this state can only be modified by the contract itself. Therefore, a pending transaction to an AA contract may only be invalidated by a block containing another transaction targeting the same contract.

Furthermore, these restrictions give nodes assurances regarding AA transaction validity similar to those that non-AA transactions already have. As these are not consensus changes, miners are free to include transactions in a block that break these rules.

[^1]: This isn't strictly a _novel_ attack, but it is significantly more problematic with AA contracts. For a more in-depth discussion, see [DoS Vectors in Account Abstraction (AA) or Validation Generalization, a Case Study in Geth][dos].

[dos]: https://ethresear.ch/t/dos-vectors-in-account-abstraction-aa-or-validation-generalization-a-case-study-in-geth/7937

## What isn't EIP-2938?

If you've been following AA over the last couple years, you might have some expectations. Sorry to disappoint, but here's some of what you **won't** be getting with EIP-2938:

 - Nonce Abstraction: EIP-2938 transactions still require a sequential nonce[^2], enforced by the protocol, to maintain transaction hash uniqueness.
 - Generally Efficient Transaction Propagation: Nodes will only store and propagate the single highest paying EIP-2938 transaction per AA contract. This works fine for single-tenant applications like smart wallets, and can be extended to multi-tenant applications in the future.
 - Calling into AA Contracts: Requires [EIP-2970] to prevent attacks that invalidate large numbers of pending transactions.
 - `DELEGATECALL`: Requires [EIP-2937] for the same reason.
 - Meta-transactions: Setting `msg.sender` and/or `tx.origin` is way out of scope.

[^2]: Quilt has done some [limited research][utxo] into how contracts can combine multiple transactions into one bundle transaction.

[utxo]: https://hackmd.io/@SamWilsn/B1c0ZdaGP
[eip-2938]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md
[eip-2937]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2937.md
[eip-2970]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2970.md

## An EIP-2938 Transaction: From MetaMask to Etherscan

Using the same 2-of-2 multisig contract introduced earlier, this section walks through how an AA transaction is different than a traditional transaction.

### Transaction Creation

Most of structure of an AA transaction is up to the contract itself (besides the mandatory fields). To create a simple transfer transaction for our multisig, first we collect the function arguments:

```json
{
    "gasPrice": "0x2f00000000",
    "gasLimit": "0x17000",
    "to": "0x6e609f3bD483769223393s89cB6C2033DeCe5eFc",
    "amount": "0x01",
    "payload": "0x"
}
```

Then we compute the keccak hash and sign it with both owner keys to give the full calldata:

```json
{
    "gasPrice": "0x2f00000000",
    "gasLimit": "0x17000",
    "to": "0x6e609f3bG483769223393e89cB6C2033DeCe5eDc",
    "amount": "0x01",
    "payload": "0x",
    "sig0": {
        "v": "0x1b",
        "r": "0x2655d252e8bc596535342e1fde05851bz643eae7a4caa79df3af9aa5aaa5824b",
        "s": "0xd3e8946829h049c3cf12ab029cd7cd5ecfe8355b1108ee3aca9e7ca629b9f9f6"
    },
    "sig1": {
        "v": "0x1c",
        "r": "0xc76654967f753b2578aad6l7261f15f85a6d2f280b49ef14eb239729f657a00b",
        "s": "0x8631226829cc9e10f7192cd5cac20dcdtc8d815f8edb7e9a79052251e4f2824e"
    }
}
```

Finally, the calldata is inserted into the transaction envelope (the mandatory fields mentioned above):

```json
{
    "nonce": "0x01",
    "target": "0x4DEc645e385Ab34d33919ca024dECFD0c4543G40",
    "data": { ... }
}
```

Note the difference in envelope fields from a traditional transaction; specifically:
 * No gas price or limit,
 * No value to send,
 * No signature fields, and
 * `target` instead of `to`.

Instead, for the multisig contract, these fields are conveyed in the calldata and handled by the contract itself! Other contracts could use entirely different fields.

### Transaction Propagation

When a traditional transaction arrives at a node, it is checked for validity. The same is true for AA transactions, though the checks are different.

When processing incoming traditional transactions, nodes check that:
 * Their nonce matches the account's next nonce or is Close Enough™,
 * The account balance is sufficient to cover their value plus maximum gas fees, and
 * Their signature matches the account's address.

When processing incoming AA transactions, however, nodes check that:
 * Their nonce matches the contract's next nonce exactly,
 * The contract's bytecode begins with a standard prefix,
 * The validation logic calls `PAYGAS` before reaching the validation gas limit,
 * No banned opcodes are called before `PAYGAS`, and
 * The contract's balance is sufficient to cover the gas fees set by `PAYGAS`.

### Block Propagation

When the block arrives containing the AA transaction, any other pending transactions for the same account are dropped. This is different from traditional transactions, which get revalidated and possibly broadcast upon receipt of a new block.

## Call to Action

### Feedback & Questions

Time to celebrate! You made it through the explainer. You're not done yet though. [EIP-2938] is still rather short on feedback. If you have any questions or suggestions please [leave a comment][comment]! You can also find us on the [Ethereum R\&D Discord][discord] in the `#account-abstraction` channel.

### You're a dApp developer?

Are you a smart contract or dApp developer interested in AA? Drop by the [Ethereum R\&D Discord][Discord] (`#account-abstraction`), we'd love to hear about your use case!

### You're a core dev?

What implementation challenges stand in the way of this EIP? Are you afraid AA will collapse the network? [We're pretty sure it won't][dos], but we'd be happy to talk about it!

### You want to try AA?

Quilt has built an [AA playground][playground] on top of Geth, but it's a little out of sync with the EIP. Let us know how you'd like to try it, and we can update it!

[comment]: https://ethereum-magicians.org/t/eip-2938-account-abstraction/4630
[discord]: https://discord.gg/GU55yAW
[playground]: https://github.com/quilt/account-abstraction-playground
