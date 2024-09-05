### Scenario Overview

As the Ethereum Layer 2 (L2) ecosystem expands, interacting across multiple chains has become increasingly cumbersome for users. Inspired by EIP-4337 (which introduces an additional abstraction layer for accounts on Ethereum), we conceptualized Omni Account, a cross-chain account abstraction layer. With Omni Account, users only need to sign EIP-712 structured UserOperation messages, enabling seamless transactions across different chains. Omni Account significantly reduces the complexity of managing accounts across multiple blockchains.

### Architecture

Similar to EIP-4337, Omni Account includes key components such as Bundlers, UserOperations, EntryPoint contracts, and user Account contracts. However, Omni Account differs in that, after users sign a UserOperation, the Bundler packages the signed UserOp into a Zero-Knowledge Proof (ZKP). The ZKP is then passed to the on-chain EntryPoint contract, which verifies the proof before invoking the user's Account to execute the UserOp. We will gradually explain the advantages of this approach and how it scales across multiple chains.

Two key processes are moved off-chain into the ZKP (Zero-Knowledge Proof) circuit:
1. Signature verification
2. User balance and nonce state management

ZKP ensures:
1. The presence of a valid user signature and corresponding UserOp
2. The nonce increments correctly, and the gas balance is accurately calculated and updated based on the gas fields in the UserOp.

Specifically, the user's gas balance and nonce are managed via a Sparse Merkle Tree (SMT), which stores key-value pairs such as:
- (user_addr, gas_balance)
- (user_addr || chain_id, nonce)

The key `(nonce || chain_id)` is used to prevent double-spending. Note that the SMT only manages the gas balance, which is used solely for gas payments. Users’ assets are managed through their individual Account contracts.

Since signature verification and state management are handled off-chain within the ZKP, the on-chain process only needs to verify the ZKP and execute the UserOp verified by the proof. **With gas balance and nonce managed off-chain**, the Bundler can sync the verified ZKP proof output across different chains, enabling the execution of chain-specific UserOps (extended with a `chainId` field in UserOp).

### Deposit and Withdraw Pathways

In addition to the UserOp execution pathway, Omni Account supports the following deposit and withdrawal flows:
1. User assets on each chain are self-custodied within their Account, allowing them to deposit and withdraw at any time.
2. The user's gas balance is secured by both the ZKP and on-chain records, referred to as **Tickets**. Specifically, users add Deposit or Withdraw Tickets on-chain, and the Bundler incorporates these Tickets into the ZKP to update the user's gas balance. The ZKP output will contain these Tickets for validation against the on-chain Tickets during submission.

