# Elegant Degradation for Account Abstraction

| Field | Value |
|-------|-------|
| **Title** | Elegant Degradation & Backwards Compatibility for Account Abstraction |
| **Author** | Mislav Javor, Filip Dujmušić, Filipp Makarov, Venkatesh Rajendran, Shunji Zhan |
| **Created** | 2026-01-10 |
| **Status** | Draft |
| **Related** | EIP-7702, ERC-5792, ERC-6900, ERC-4337, ERC-7579 |

---

## Abstract

This document proposes a technique for extending Smart Account capabilities to users of standard Externally Owned Accounts (EOAs) without requiring EIP-7702 upgrades, fund migration, or wallet-level support. By appending a commitment hash to standard ERC-20 `approve` calldata, users can authorize complex multi-step and cross-chain operations through a single signature. The Smart Account then pulls funds, executes the committed operations, and returns resulting assets to the original EOA—all verified on-chain.

## Motivation

### The Adoption Gap

Account Abstraction promises significant UX improvements: batch execution, gas sponsorship, cross-chain operations, and scheduled transactions. However, adoption faces three persistent barriers:

1. **Fund Migration Resistance** — Users with established EOAs have transaction history, tokens across multiple chains, POAPs, soulbound tokens, and non-transferrable DeFi positions tied to their address. Migrating funds to a separate Smart Account bifurcates their on-chain identity.

2. **EIP-7702 Upgrade Friction** — While EIP-7702 allows in-place EOA upgrades, user adoption remains low due to inertia and security concerns.

3. **Wallet Support Fragmentation** — Many prominent wallets have not implemented EIP-7702. Some wallet providers lack economic incentive to do so, as advanced app-level features (gasless bridges, one-click swaps) compete with their own revenue-generating services.

### The Developer Burden

Application developers must currently maintain two separate transaction pipelines: one for Smart Account users and one for EOA users. This increases development complexity and fragments the user experience.

### Design Goals

This proposal aims to:

- Enable Smart Account features for **all** EOA users regardless of wallet support
- Require **zero opt-in** from wallets, apps, or infrastructure providers
- Maintain a **familiar UX** (standard approval flow)
- Preserve **on-chain verification** of all operations
- Provide a **migration path** toward full Account Abstraction adoption

## Specification

### Overview

The technique exploits a property of EVM calldata encoding: bytes appended beyond the expected length of a function call are ignored by the target contract but remain accessible on-chain. This allows embedding a commitment to additional operations within a standard `approve` transaction.

The user signs an `approve` transaction which approves some amount of some token to their Smart Account. The `approve` call appends the merkle tree root hash of all the function calls which need to happen after the funds are pulled to the smart account. Usually the last operation is returning any resulting funds back to the user EOA. The smart account uses `transferFrom` to consume the approval and execute all additional actions. 

An example:

User approves 3000 USDC to their Smart Account. The `approve` call has a hash which encodes the following instructions:

1. `transferFrom` which pulls 3000 USDC from the user EOA to their SCA
2. `swap` call which swaps USDC into WETH
3. `deposit` call which deposits the WETH into a Vault
4. `withdraw` call which pulls the vault tokens back to the user EOA

This is a single-signature batch execution from a regular user EOA!

### Calldata Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                     Standard ERC-20 Approve                     │
├─────────────────────────────────────────────────────────────────┤
│ 0x095ea7b3 (approve selector)                                   │
│ <spender: smart_account_address> (32 bytes)                     │
│ <amount: uint256> (32 bytes)                                    │
├─────────────────────────────────────────────────────────────────┤
│                     Appended Commitment                         │
├─────────────────────────────────────────────────────────────────┤
│ <operations_hash: bytes32> (Merkle root of operation set)       │
└─────────────────────────────────────────────────────────────────┘
```

### Execution Flow

```
┌──────────┐     1. Sign approve + hash      ┌──────────┐
│          │ ──────────────────────────────► │          │
│   User   │                                 │  Relayer │
│  (EOA)   │ ◄────────────────────────────── │          │
│          │     6. Receive result tokens    │          │
└──────────┘                                 └────┬─────┘
                                                  │
                                    2. Submit     │    3. Execute
                                       approval   │       operations
                                                  ▼
                                          ┌──────────────┐
                                          │              │
                                          │    Smart     │
                                          │   Account    │
                                          │              │
                                          └──────┬───────┘
                                                 │
                         ┌───────────────────────┼───────────────────────┐
                         │                       │                       │
                         ▼                       ▼                       ▼
                   ┌──────────┐           ┌──────────┐           ┌──────────┐
                   │ Chain A  │           │ Chain B  │           │ Chain N  │
                   │ (Source) │           │ (Bridge) │           │ (Target) │
                   └──────────┘           └──────────┘           └──────────┘
```

**Step-by-step:**

1. **User signs** a standard ERC-20 `approve` call with the operations hash appended to calldata
2. **Relayer submits** the approval transaction to the source chain
3. **Smart Account pulls** approved tokens from the user's EOA via `transferFrom`
4. **Smart Account verifies** that the relayer-provided operations match the signed hash (Merkle proof validation)
5. **Smart Account executes** the operation set (may span multiple chains)
6. **Smart Account transfers** resulting assets back to the user's EOA

### Hash Commitment Structure

Operations are encoded as a Merkle tree, allowing efficient on-chain verification:

```
                    Root Hash (signed by user)
                           /    \
                          /      \
                    Hash(A,B)   Hash(C,D)
                      / \         / \
                     A   B       C   D
                     
Where A, B, C, D = keccak256(chainId, target, value, calldata, nonce)
```

### On-Chain Validation: TxValidatorLib

The critical security property of this proposal is that **all validation happens on-chain**. The Smart Account does not blindly trust the relayer—it reconstructs and verifies the user's original signature against the committed operations.

#### How It Works

The `TxValidatorLib` library performs on-chain validation by:

1. **Parsing the raw signed transaction** — The relayer submits the fully signed EVM transaction (Legacy or EIP-1559) as proof
2. **Reconstructing the unsigned transaction hash** — The library strips the signature and recalculates the transaction hash
3. **Recovering the signer** — Using `ecrecover` on the reconstructed hash to verify the EOA actually signed this transaction
4. **Extracting the commitment** — Reading the appended bytes32 (superTxHash) from the transaction's calldata
5. **Verifying Merkle inclusion** — Confirming the current operation is part of the Merkle tree the user committed to

#### Transaction Data Structure

The signed transaction contains the following appended data:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Signed EVM Transaction                        │
├─────────────────────────────────────────────────────────────────┤
│ [0]        txType (0x00 = Legacy, 0x02 = EIP-1559)              │
│ [1..n]     RLP-encoded transaction fields                       │
├─────────────────────────────────────────────────────────────────┤
│                   Appended Validation Data                      │
├─────────────────────────────────────────────────────────────────┤
│ [n+1..]    Merkle proof items (32 bytes each)                   │
│ [...]      proofItemsCount (1 byte)                             │
│ [...]      lowerBoundTimestamp (6 bytes)                        │
│ [...]      upperBoundTimestamp (6 bytes)                        │
└─────────────────────────────────────────────────────────────────┘
```

#### Core Validation Logic

```solidity
function validateUserOp(
    bytes32 userOpHash,
    bytes calldata parsedSignature,
    address expectedSigner
)
    internal
    view
    returns (uint256)
{
    // 1. Decode the full signed transaction
    TxData memory decodedTx = decodeTx(parsedSignature);

    // 2. Compute the MEE UserOp hash (includes timestamp bounds)
    bytes32 meeUserOpHash = MEEUserOpHashLib.getMEEUserOpHash(
        userOpHash, 
        decodedTx.lowerBoundTimestamp, 
        decodedTx.upperBoundTimestamp
    );

    // 3. Verify the EOA actually signed this transaction
    bytes memory signature = abi.encodePacked(decodedTx.r, decodedTx.s, decodedTx.v);
    if (!EcdsaHelperLib.isValidSignature(expectedSigner, decodedTx.utxHash, signature)) {
        return SIG_VALIDATION_FAILED;
    }

    // 4. Verify the operation is in the committed Merkle tree
    if (!MerkleProofLib.verify(decodedTx.proof, decodedTx.superTxHash, meeUserOpHash)) {
        return SIG_VALIDATION_FAILED;
    }

    // 5. Return validation data with timestamp bounds
    return _packValidationData(false, decodedTx.upperBoundTimestamp, decodedTx.lowerBoundTimestamp);
}
```

#### Unsigned Transaction Hash Reconstruction

The library supports both Legacy (Type 0) and EIP-1559 (Type 2) transactions:

```solidity
function calculateUnsignedTxHash(
    uint8 txType,
    bytes memory rlpEncodedTx,
    uint256 rlpEncodedTxPayloadLen,
    uint256 v,
    bytes32 r,
    bytes32 s
) private pure returns (bytes32 hash) {
    // Strip signature from RLP-encoded transaction
    uint256 totalSignatureSize = 
        uint256(r).encodeUint().length + 
        uint256(s).encodeUint().length + 
        v.encodeUint().length;
    
    bytes memory rlpEncodedTxNoSig = rlpEncodedTx.slice(
        totalPrefixSize, 
        rlpEncodedTx.length - totalSignatureSize - totalPrefixSize
    );
    
    if (txType == EIP1559_TX_TYPE) {
        // EIP-1559: hash(0x02 || RLP([...fields]))
        return EfficientHashLib.hash(
            abi.encodePacked(txType, prependRlpContentSize(rlpEncodedTxNoSig, ""))
        );
    } else if (txType == LEGACY_TX_TYPE) {
        // Legacy with EIP-155: hash(RLP([...fields, chainId, 0, 0]))
        if (v >= EIP_155_MIN_V_VALUE) {
            return EfficientHashLib.hash(
                prependRlpContentSize(
                    rlpEncodedTxNoSig,
                    abi.encodePacked(
                        uint256(_extractChainIdFromV(v)).encodeUint(),
                        uint256(0).encodeUint(),
                        uint256(0).encodeUint()
                    )
                )
            );
        } else {
            // Legacy pre-EIP-155
            return EfficientHashLib.hash(prependRlpContentSize(rlpEncodedTxNoSig, ""));
        }
    }
}
```

#### EIP-1271 Support

The library also supports signature validation for smart contract wallets via EIP-1271:

```solidity
function validateSignatureForOwner(
    address expectedSigner,
    bytes32 dataHash,
    bytes calldata parsedSignature
) internal view returns (bool) {
    TxDataShort memory decodedTx = decodeTxShort(parsedSignature);

    // Verify EOA signature
    bytes memory signature = abi.encodePacked(decodedTx.r, decodedTx.s, decodedTx.v);
    if (!EcdsaHelperLib.isValidSignature(expectedSigner, decodedTx.utxHash, signature)) {
        return false;
    }

    // Verify Merkle inclusion
    if (!MerkleProofLib.verify(decodedTx.proof, decodedTx.superTxHash, dataHash)) {
        return false;
    }
    return true;
}
```

#### Why This Matters

The on-chain validation approach provides several critical guarantees:

| Property | Guarantee |
|----------|-----------|
| **Trustless** | Relayer cannot forge signatures or alter operations |
| **Verifiable** | Anyone can verify the proof on-chain |
| **Replay-protected** | Timestamp bounds and nonces prevent replay attacks |
| **Chain-aware** | EIP-155 chain ID extraction prevents cross-chain replay |

### Smart Account Validation Module

The Smart Account module integrates `TxValidatorLib` to validate operations:

```solidity
function executeWithCommitment(
    bytes32 signedHash,
    Operation[] calldata operations,
    bytes32[] calldata merkleProofs
) external {
    // 1. Verify operations match the signed commitment
    bytes32 computedRoot = computeMerkleRoot(operations, merkleProofs);
    require(computedRoot == signedHash, "Commitment mismatch");
    
    // 2. Pull funds from EOA (approval already granted)
    IERC20(token).transferFrom(userEOA, address(this), amount);
    
    // 3. Execute operations
    for (uint i = 0; i < operations.length; i++) {
        _execute(operations[i]);
    }
    
    // 4. Return resulting funds to EOA
    _sweepToEOA(userEOA);
}
```

### Native Token (ETH) Support

The ERC-20 `approve` pattern does not apply to native ETH, as ETH has no approval mechanism. To support operations that require ETH as input, we introduce a lightweight **EtherForwarder** contract.

#### The Problem

Unlike ERC-20 tokens, native ETH cannot be "approved" to a spender. The user must explicitly send ETH in a transaction. This breaks the single-signature model for ETH-based operations.

#### The Solution

A stateless forwarder contract that:
1. Receives ETH from the user (with commitment hash appended to calldata)
2. Immediately forwards the ETH to the user's Smart Account
3. The Smart Account then executes the committed operations as normal

#### Execution Flow (Native ETH)

```
┌──────────┐   1. forward(smartAccount)    ┌───────────────┐
│          │      + ETH + hash             │               │
│   User   │ ─────────────────────────────►│ EtherForwarder│
│  (EOA)   │                               │               │
└──────────┘                               └───────┬───────┘
     ▲                                             │
     │                                   2. Forward ETH
     │                                             │
     │                                             ▼
     │                                     ┌──────────────┐
     │        5. Return result tokens      │              │
     └─────────────────────────────────────│    Smart     │
                                           │   Account    │
                                           │              │
                                           └──────┬───────┘
                                                  │
                                        3. Verify hash
                                        4. Execute ops
```

#### EtherForwarder Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

/**
 * @title EtherForwarder
 * @notice A contract that forwards received Ether to a specified address
 * @author Biconomy
 */
contract EtherForwarder {
    error ZeroAddress();
    error ForwardFailed();

    /**
     * @notice Forwards the received Ether to the specified destination address
     * @param destination The address to forward the Ether to
     */
    function forward(address destination) external payable {
        if (destination == address(0)) revert ZeroAddress();

        // Forward the Ether using assembly
        bool success;
        assembly {
            // Gas-efficient way to forward ETH
            success := call(
                gas(),        // Forward all available gas
                destination,  // Destination address
                callvalue(),  // Amount of ETH to send
                0,            // No data to send
                0,            // No data size
                0,            // No data to receive
                0             // No data size to receive
            )
        }

        if (!success) revert ForwardFailed();
    }

    /**
     * @notice Prevents accidental Ether transfers without a destination
     */
    receive() external payable {
        revert("Use forward() function to send Ether");
    }

    /**
     * @notice Prevents accidental Ether transfers without a destination
     */
    fallback() external payable {
        revert("Use forward() function to send Ether");
    }
}
```

#### Design Rationale

| Property | Benefit |
|----------|---------|
| **Stateless** | No storage reads/writes; minimal gas overhead |
| **Singleton** | One deployment per chain; all users share the same forwarder |
| **No approval required** | ETH is forwarded atomically in a single call |
| **Hash appended to calldata** | Same commitment pattern as ERC-20 flow |
| **Receive/fallback rejection** | Prevents accidental ETH loss from direct transfers |

#### Calldata Structure (Native ETH)

```
┌─────────────────────────────────────────────────────────────────┐
│                     EtherForwarder.forward()                    │
├─────────────────────────────────────────────────────────────────┤
│ 0x... (forward selector)                                        │
│ <destination: smart_account_address> (32 bytes)                 │
├─────────────────────────────────────────────────────────────────┤
│                     Appended Commitment                         │
├─────────────────────────────────────────────────────────────────┤
│ <operations_hash: bytes32> (Merkle root of operation set)       │
└─────────────────────────────────────────────────────────────────┘
│ msg.value: ETH amount                                           │
└─────────────────────────────────────────────────────────────────┘
```

This allows the same commitment-based execution model to work for both ERC-20 tokens and native ETH, maintaining a consistent UX regardless of the input asset type.

### Atomicity Guarantees

| Scenario | Guarantee |
|----------|-----------|
| Single-chain operations | Fully atomic (single transaction post-approval) |
| Cross-chain operations | Per-chain atomicity; cross-chain atomicity follows standard bridging assumptions |

## Rationale

### Why Merkle Tree Commitment?

A Merkle tree allows:
- Compact on-chain verification (O(log n) proof size)
- Selective disclosure of operations (notably in multi-chain environments, you only need to post the proof of the action for that chain)
- Extensibility for future operation types
- Ability to encode operations which span multiple chains

### Why Return Funds to EOA?

Returning resulting assets to the original EOA maintains the user's mental model of "operating from their existing wallet." This is critical for adoption—users see their balances change in their familiar interface.

## Backwards Compatibility

This proposal is fully backwards compatible:

- **ERC-20 contracts**: Ignore appended calldata; function as normal
- **Existing wallets**: Display standard approval UI
- **Block explorers**: Transaction decodes as normal approval
- **Smart Account standards**: Compatible with ERC-7579 and ERC-6900 via validator module

No changes to existing infrastructure are required.

## Security Considerations

### Commitment Integrity

The on-chain Merkle root verification ensures relayers cannot execute operations other than those explicitly signed by the user. Attempted deviation results in transaction revert.

### Approval Scope

The user only approves the specific token amount required for the operation. Standard ERC-20 approval risks apply—users should verify the spender address (their Smart Account).

### Relayer Trust Model

Relayers are **untrusted**. They can:
- Delay execution (liveness concern, not security)
- Fail to execute (user retains funds in EOA)

Relayers **cannot**:
- Execute different operations (commitment mismatch reverts)
- Steal funds (operations are verified on-chain)

### Self Relaying

As part of the Trustlessness efforts spearheaded by EF, this proposal satisfies the following properties:

- User can self-commit transactions
- Wallets can relay on users behalf
- All validation is done onchain

### Phishing Considerations

As noted in the `TxValidatorLib` documentation:

> In theory, the last 32 bytes of calldata from any transaction by the EOA can be interpreted as a superTx hash. Even if it was not assumed. This introduces the potential risk of phishing attacks where the user may unknowingly sign a transaction where the last 32 bytes of the calldata end up being a superTx hash.

**Mitigation**: Wallets and users should be aware of this risk and should not sign transactions where the last 32 bytes of the calldata do not belong to the function arguments. Clear signing standards (EIP-7730) can help surface this to users.

### Hardware Wallet Compatibility

Some hardware wallets reject transactions with unexpected calldata length. Firmware updates may be required for full compatibility. Unlike software wallets, hardware wallet manufacturers do not have competing economic incentives against supporting this feature.

### Clear Signing

Users do not see human-readable operation details for the appended hash. Integration with [EIP-7730 (Clear Signing)](https://eips.ethereum.org/EIPS/eip-7730) can address this limitation by enabling wallets to display decoded operation sequences.

### EtherForwarder Security

The EtherForwarder contract is intentionally minimal to reduce attack surface:

- **No state**: Cannot be exploited via storage manipulation
- **No owner**: No privileged functions; immutable after deployment
- **Atomic forwarding**: ETH is forwarded in the same transaction; no funds are held
- **Explicit destination**: User signs the destination address; cannot be redirected

### Timestamp Bounds

The `TxValidatorLib` includes timestamp validation (`lowerBoundTimestamp`, `upperBoundTimestamp`) to:
- Prevent indefinite validity of signed operations
- Allow time-bounded execution windows
- Enable scheduled/delayed execution patterns

## Reference Implementation

### Live Deployment

This technique is deployed in production as part of Biconomy MEE.

**Example Transaction**: Cross-chain batch execution (Base → Arbitrum)

| Step | Description | Chain |
|------|-------------|-------|
| 1 | Approve USDe + commit hash | Base |
| 2 | Pull USDe, bridge to Arbitrum | Base |
| 3 | Receive USDe, supply to Compound | Arbitrum |
| 4 | Return Compound vault tokens to EOA | Arbitrum |

**Explorer Links:**
- Execution Details: [meescan.biconomy.io](https://meescan.biconomy.io/details/0x617c16a2071e4b109f6575ccaa9b1560b3cf8f6dec8c555a5db71484b9c12248)
- Polygon Transaction: [polygonscan.com](https://polygonscan.com/tx/0xd3030f83185e7f295f675efd92a457e7b199d803ca553244f30a1019f95ae72b)
- Base Transaction: [basescan.org](https://basescan.org/tx/0x32f6def621dd916c9d76afb2d87de05c321b2314e117213aeb5cbec2412955b9)

**Video Demonstration:**  
https://www.youtube.com/watch?v=2QOPEPjTcjU

### Source Code

- TxValidatorLib: [GitHub](https://github.com/bcnmy) (Biconomy MEE repository)
- EtherForwarder: Included in this document

### Integration Guide

This approach integrates with existing Account Abstraction infrastructure, including:

- ERC-7579 Accounts 
- ERC-6900 Accounts 

## Not A Replacement

This proposal _doesn't_ serve as a replacement for initiatives such as EIP-7701, EIP-7702, ERC-5792, ERC-7715, ERC-7710, etc... It serves as an elegant degradation path for users who _don't_ have smart account wallets.

## Appendix: Comparison with Alternatives

| Approach | Single Sig | No Migration | No Wallet Changes | Works Today |
|----------|:----------:|:------------:|:-----------------:|:-----------:|
| Native EOA | ✓ | ✓ | ✓ | ✓ |
| Smart Account (new) | ✓ | ✗ | ✓ | ✓ |
| EIP-7702 | ✓ | ✓ | ✗ | Partial |
| **This Proposal** | ✓ | ✓ | ✓ | ✓ |

## Open Questions

1. **Standardization**: Should the commitment hash format be standardized for cross-provider interoperability?
2. **Multi-Token Operations**: How should operations requiring multiple input tokens be handled?

## Acknowledgements

We thank the broader Account Abstraction research community for ongoing discussions on user onboarding challenges.

---

**Discussion**: We invite feedback from researchers, wallet developers, and application builders. This document represents one solution to the backwards compatibility problem—alternative approaches are welcome.
