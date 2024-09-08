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

The key `(nonce || chain_id)` is used to prevent replay attack. Note that the SMT only manages the gas balance, which is used solely for gas payments. Users’ assets are managed through their individual Account contracts.

Since signature verification and state management are handled off-chain within the ZKP, the on-chain process only needs to verify the ZKP and execute the UserOp verified by the proof. **With gas balance and nonce managed off-chain**, the Bundler can sync the verified ZKP proof output across different chains, enabling the execution of chain-specific UserOps (extended with a `chainId` field in UserOp).

### Deposit and Withdraw Pathways

In addition to the UserOp execution pathway, Omni Account supports the following deposit and withdrawal flows:
1. User assets on each chain are self-custodied within their Account, allowing them to deposit and withdraw at any time.
2. The user's gas balance is secured by both the ZKP and on-chain records, referred to as **Tickets**. Specifically, users add Deposit or Withdraw Tickets on-chain, and the Bundler incorporates these Tickets into the ZKP to update the user's gas balance. The ZKP output will contain these Tickets for validation against the on-chain Tickets during submission.


### Tech Stack + Core Data Structures

- **ZKP**: Currently, we are using the fastest zkVM, [sp1](https://github.com/succinctlabs/sp1), as an initial implementation of the ZKP circuit logic. In the future, we plan to migrate to a more specialized ZK language for better performance.
- **Frontend**: Built from a TypeScript template using Create React App, enhanced with Chakra UI for UI components, Ethers.js for blockchain interactions.
- **Backend**: This Go-based implementation serves as a basic EIP-4337 bundler, responsible for bundling user operations and interacting with L1 and ZKP systems. It handles the aggregation and submission of transactions while ensuring secure and efficient communication with the underlying layers.
- **Smart Contracts**: Our smart contracts are developed by extending an open-source contract library. We have streamlined the original implementation by removing unnecessary business modules, replaced the original deposit module, and added new cross-chain functionality using LayerZeroV2 and zk verification modules.

### LayerZero for Cross-Chain Communication
 With gas balance and nonce managed off-chain, the Bundler need to sync the verified ZKP proof output across different chains. We utilize LayerZero as the cross-chain protocol due to its stability and permissionless nature. This ensures reliable, decentralized cross-chain operations, allowing seamless interactions across multiple blockchains.

### SMT Data Structure
Our Sparse Merkle Tree (SMT) implementation is inspired by this article: [A Hacker’s Guide to Layer 2 Zero Merkle Trees](https://medium.com/@carterfeldman/a-hackers-guide-to-layer-2-zero-merkle-trees-from-scratch-d612ea846016). However, we will later improve it with a more efficient SMT that requires fewer hash operations and uses a more ZK-friendly hashing algorithm.

Here’s a two-step overview of the Zero Sparse Merkle Tree we currently use:

1. The tree guarantees the immutability of the stored (k,v) pairs, where `k` is the index of the tree's leaf node.
2. It cryptographically maps the user’s state (gas balance, nonce) to a (k,v) pair that can be stored in the SMT in a collision-resistant manner.

#### Step 1: Zero Sparse Merkle Tree
The Zero Sparse Merkle Tree is a complete binary tree with a height of 256. Initially, all leaf nodes are set to 0, referred to as `z0`. The parent node, `z1`, is `hash(z0, z0)`, and this pattern continues upward until the root node, `z256`, is formed as `hash(z255, z255)`. By storing just 256 hash values, we can initialize a tree with 2^256 leaf nodes.

When inserting a value into the tree using `set_leaf(index, value)`, the binary representation of `index` uniquely determines a hash path. The value is inserted at the specified `index`, and hash operations are performed along the path, updating the SMT root by hashing with the left and right siblings. Due to the cryptographic security of the hash function and the uniqueness of the hash path, this SMT ensures the immutability of stored (k, v) pairs. The storage complexity for this SMT is O(P), where P is the number of (k, v) pairs, making it sparse. In ZKP, SMT requires 512 hash operations for a single leaf update.

#### Step 2: Mapping User State to (k, v) Pairs
- **For gas balance**:  
   `k = sha256(identifier || user_addr || 0)`  
   The identifier for balance is 0. `v` is the balance value in hexadecimal.
   
- **For nonce**:  
   `k = sha256(identifier || user_addr || chain_id || 0)`  
   The identifier for nonce is 1, and different `chain_id` values ensure nonce states are maintained separately to prevent cross-chain replay attacks. `v` is the nonce value in hexadecimal.

Through these two steps, we can securely and immutably manage the user's gas balance and nonce state off-chain within the ZKP.

### UserOperation Data Structure
```solidity
struct UserOperation {
    address sender;
    uint256 nonce;
    uint64 chainId;
    bytes initCode;
    bytes callData;
    uint256 callGasLimit;
    uint256 verificationGasLimit;
    uint256 preVerificationGas;
    uint256 maxFeePerGas;
    uint256 maxPriorityFeePerGas;
    bytes paymasterAndData;
}
```
Compared to EIP-4337, we have added a `uint64 chainId` field. Users sign this EIP-712 structured message to generate a signature.

### Ticket Data Structure
```solidity
struct Ticket {
    address user;
    uint256 amount;
    uint256 timestamp;
}
```
The hash of the abi_encoded `Ticket` is stored on-chain. The ZKProof output contains the `Ticket` and the EntryPoint contract will compare it with the on-chain hash for verification.
