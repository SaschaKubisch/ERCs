---
eip: 7xxx
title: ONCHAINID - An Onchain Identity System
description: Formalizing ONCHAINID, a self-sovereign identity system on Ethereum with a minimal Key Management System (KMS) by default and optional custom permissions logic.
author: Joachim Lebrun (@Joachim-Lebrun), Kevin Thizy (@Nakasar), Matthew Pinnock (@AureliusEth), Luc Falempin (@lfalempin), Tony Malghem (@TonyMalghem)
discussions-to: //TBD
status: Draft
type: Standards Track
category: ERC
created: 2024-xx-xx
---

## Abstract

ONCHAINID is a blockchain-based identity system that combines the functionalities of a key manager and a claim holder. The key manager holds keys to sign actions and execute instructions, while the claim holder manages claims that can be attested by third parties or self-attested. The identity contract defined by ONCHAINID `IIdentity` integrates these functionalities, providing a comprehensive solution for individuals and organizations to enforce compliance and access digital assets or protocols with permission mechanisms.

ONCHAINID introduces a minimal Key Management System (KMS) by default, where all keys have full permissions. Additionally, users can optionally implement custom permissions logic, dynamically assigning specific permissions to each key.

## Motivation

The motivation behind ONCHAINID is to address the inadequacies of the existing Ethereum protocol in managing complex account structures and verifying onchain claims about an identity. The current protocol lacks a standardized way for DApps and smart contracts to check the claims about an identity, and there is no formalized way to manage these claims onchain. Moreover, the protocol does not provide a comprehensive solution for managing keys associated with an account.

ONCHAINID aims to provide a self-sovereign identity system on the blockchain that allows users to create and manage their own identities. This identification solution enables compliance and identity verifications within the pseudonymous framework of public blockchain networks. It integrates the functionalities of a key manager and a claim holder, as proposed in the Key Manager and Claim Holder proposals by Fabian Vogelsteller. However, these proposals were never formalized as EIPs, remaining at the issue state. This EIP aims to formalize these proposals and integrate them into the ONCHAINID system.

## Specification

### Key Management

ONCHAINID provides a flexible key management system where keys are cryptographic public keys or contract addresses that have permission to operate the identity or to interact with services on its behalf.

#### Default Minimal Key Management System (KMS)

- By default, each key added to the identity has full permissions. There are no predefined key purposes; instead, any added key can manage the identity, execute actions, or manage claims without restrictions.
- This approach simplifies the system for users who do not require fine-grained control.

#### Custom Permissions Logic (Optional)

- Optionally, users can assign specific permissions to keys.
- Permissions are represented as `bytes32` values, and each key can have a list of permissions assigned to it.
- A key without any assigned permissions is considered to have full control over the identity.
- Permissions can be dynamically assigned or revoked by the identity owner.

The structure should be as follows:

- `key`: A public key owned by this identity
  - `permissions`: `bytes32[]` List of permissions assigned to the key (empty list implies full control)

```solidity
struct Key {
    address keyAddress;  // Address of the key holder
    bool active;         // Whether the key is active
    bytes32[] permissions; // List of permissions (optional)
}
```

#### Permissions List

Define permissions for all possible function calls of the ONCHAINID contract and inherited contracts:

- `ADD_KEY`: Permission to add new keys.
- `REMOVE_KEY`: Permission to remove keys.
- `ASSIGN_PERMISSION`: Permission to assign a permission to a key.
- `REVOKE_PERMISSION`: Permission to revoke a permission from a key.
- `ADD_CLAIM`: Permission to add claims.
- `REMOVE_CLAIM`: Permission to remove claims.
- `EXECUTE`: Permission to execute transactions on behalf of the identity.
- `APPROVE`: Permission to approve executions or claims.
- **Custom permissions**: Users can also define their own permissions as needed for external contracts that integrate with ONCHAINID.

These permissions are represented as `bytes32` constants:

```solidity
bytes32 constant ADD_KEY = keccak256("ADD_KEY");
bytes32 constant REMOVE_KEY = keccak256("REMOVE_KEY");
bytes32 constant ASSIGN_PERMISSION = keccak256("ASSIGN_PERMISSION");
bytes32 constant REVOKE_PERMISSION = keccak256("REVOKE_PERMISSION");
bytes32 constant ADD_CLAIM = keccak256("ADD_CLAIM");
bytes32 constant REMOVE_CLAIM = keccak256("REMOVE_CLAIM");
bytes32 constant EXECUTE = keccak256("EXECUTE");
bytes32 constant APPROVE = keccak256("APPROVE");
```

#### Key Management Functions

- **Add Key**: Adds a new key to the identity with full permissions by default. Only the owner can add keys or keys with the `ADD_KEY` permission.
  ```solidity
  function addKey(address _keyAddress) public hasPermission(ADD_KEY) {
      // Logic for adding the key
      keys[_keyAddress] = Key({
          keyAddress: _keyAddress,
          active: true,
          permissions: new bytes32[](0) // Default to full permissions
      });
      emit KeyAdded(_keyAddress);
  }
  ```
- **Remove Key**: Removes an existing key from the identity. Only the owner or keys with the `REMOVE_KEY` permission can remove keys.
  ```solidity
  function removeKey(address _keyAddress) public hasPermission(REMOVE_KEY) {
      // Logic for removing the key
      delete keys[_keyAddress];
      emit KeyRemoved(_keyAddress);
  }
  ```
- **Assign Permission**: Assigns a custom permission to a key (optional). Only the owner or keys with the `ASSIGN_PERMISSION` permission can assign permissions.
  ```solidity
  function assignPermission(address _keyAddress, bytes32 _permission) public hasPermission(ASSIGN_PERMISSION) {
      // Logic for assigning permission
      keys[_keyAddress].permissions.push(_permission);
      emit PermissionAssigned(_keyAddress, _permission);
  }
  ```
- **Revoke Permission**: Revokes a custom permission from a key. Only the owner or keys with the `REVOKE_PERMISSION` permission can revoke permissions.
  ```solidity
  function revokePermission(address _keyAddress, bytes32 _permission) public hasPermission(REVOKE_PERMISSION) {
      // Logic for revoking permission
      bytes32[] storage permissions = keys[_keyAddress].permissions;
      for (uint256 i = 0; i < permissions.length; i++) {
          if (permissions[i] == _permission) {
              permissions[i] = permissions[permissions.length - 1];
              permissions.pop();
              emit PermissionRevoked(_keyAddress, _permission);
              break;
          }
      }
  }
  ```

#### Permission Enforcement

- **Modifier for Permission Check**: Use a modifier to enforce permission checks before executing functions.

```solidity
modifier hasPermission(bytes32 _permission) {
    require(
        keys[msg.sender].active && (keys[msg.sender].permissions.length == 0 || hasSpecificPermission(msg.sender, _permission)),
        "Insufficient permissions"
    );
    _;
}

function hasSpecificPermission(address _keyAddress, bytes32 _permission) internal view returns (bool) {
    bytes32[] memory permissions = keys[_keyAddress].permissions;
    for (uint256 i = 0; i < permissions.length; i++) {
        if (permissions[i] == _permission) {
            return true;
        }
    }
    return false;
}
```

### Claim Management

#### Claim Storage

An identity contract can hold claims. Claims are structured data that bear information, being by the content of the `data` property or by its sole existence.

A claim is structured as follows:

```solidity
struct Claim {
  uint256 topic;
  uint256 scheme;
  address issuer;
  bytes signature;
  bytes data;
  string uri;
}
```

| Field       | Description                                                                                                                               |
|-------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `topic`     | An integer representing the subject of the claim.                                                                                         |
| `scheme`    | An integer specifying the format of the claim data property.                                                                              |
| `issuer`    | The contract address of the Claim Issuer that issued the claim (for self-attested claims, this will be the identity contract address).    |
| `signature` | The signature of the claim by the claim issuer (signature method is not standardized).                                                    |
| `data`      | Data property of the claim (digest of the data is included in the signature). The content is formatted according to the specified scheme. |
| `uri`       | Claims may reference an external storage (IPFS, API, etc) for additional content.                                                         |

> This specification does not cover the algorithm and methods for claim signature. Each claim issuer may use different keys format.

#### Adding a Claim

To store or update a claim on the Identity, the method `addClaim()` is called.

This method MUST emit a `ClaimAdded` event when a claim is stored, and `ClaimChanged` when a claim is updated.

```solidity
/**
 * @dev Emitted when a claim was added.
 *
 * Specification: MUST be triggered when a claim was successfully added.
 */
event ClaimAdded(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data, 
  string uri
);
```

```solidity
/**
 * @dev Emitted when a claim was changed.
 *
 * Specification: MUST be triggered when addClaim was successfully called on an existing claimId.
 */
event ClaimChanged(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data,
  string uri
);
```

An Identity must store only one claim of a given topic for a claim issuer. Attempting to add another claim with the same pair of topic and issuer MUST result in replacing the existing claim with the new one (and emits the `ClaimChanged` instead of the `ClaimAdded`).

Only the owner or keys with the `ADD_CLAIM` permission can add claims.

```solidity
function addClaim(...) public hasPermission(ADD_CLAIM) {
    // Logic for adding a claim
}
```

#### Removing a Claim

To remove a claim from the Identity, the method `removeClaim()` is called.

The method MUST emit a `ClaimRemoved` event.

```solidity
/**
 * @dev Emitted when a claim was removed.
 *
 * Specification: MUST be triggered when removeClaim was successfully called.
 */
event ClaimRemoved(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data,
  string uri
);
```

Only the owner or keys with the `REMOVE_CLAIM` permission can remove claims.

```solidity
function removeClaim(bytes32 _claimId) public hasPermission(REMOVE_CLAIM) {
    // Logic for removing a claim
}
```

#### Accessing Claims

Identity contracts MUST expose methods to retrieve claims.

To retrieve a claim by its ID, the method `getClaim()` is called.
The ID of the claim is computed with `keccak256(abi.encode(issuer, topic))`. One issuer can have at most one active claim of a given topic for an identity.
This method MUST return the complete claim structure:

```solidity
/**
 * @dev Get a claim by its ID.
 *
 * Claim IDs are generated using `keccak256(abi.encode(address issuer_address, uint256 topic))`.
 */
function getClaim(bytes32 _claimId)
external view returns(
    uint256 topic,
    uint256 scheme,
    address issuer,
    bytes memory signature,
    bytes memory data,
    string memory uri
);
```

For convenience, the method `getClaimIdsByTopic()` is introduced as a mandatory implementation.
This method MUST return an array of claim IDs for a given topic.

```solidity
/**
 * @dev Returns an array of claim IDs by topic.
 */
function getClaimIdsByTopic(uint256 _topic) external view returns(bytes32[] memory claimIds);
```

> An identity MAY not send all claims of a given topics but only a subset of them. An implementation COULD let the identity owner select only a few claims to be presented each time.

#### Self-attested Claims

Claims are usually issued by third parties about an identity, but some claims are intended to be created by the identity owner themselves. These claims are called self-attested claims.

Self-attested claims addition, updates and removals MUST trigger the `ClaimAdded`, `ClaimChanged` and `ClaimRemoved` events, and they SHOULD be managed by the same methods `addClaim` and `removeClaim`.

Self-attested claims use the same validity check regarding signatures (algorithms are left to the appreciation of implementers). Specifications do not include a revoke mechanism, however the Identity contract exposes a `isClaimValid` method that MUST verify the claim signature and return true if it is valid, or false otherwise.

The `isClaimValid` method MAY implement a revocation behavior based on timestamp, revocation lists or any other method deemed necessary by the implementer, returning true or false depending on the claim validity status.

```solidity
/**
 * @dev Checks if a claim is valid.
 * @param identity the identity contract related to the claim.
 * @param claimTopic the claim topic of the claim
 * @param sig the signature of the claim
 * @param data the data field of the claim
 * @return claimValid true if the claim is valid, false otherwise
 */
function isClaimValid(
    address identity,
    uint256 claimTopic,
    bytes calldata sig,
    bytes calldata data)
external view returns (bool);
```

### Identity Usage

Apart from keys and claims management, the Identity contract specified in ONCHAINID allows for execution requests and approvals. A contract (or a signer) may create a new execution request by calling the `execute()` method. Execute may be native transfers of value from the identity contract balance to another address, or execution of methods on other contracts. When an execution request is created, it awaits approval from an authorized key. Upon validation, the operation is performed.

This behavior allows Identity to hold tokens or to act as signers or simplified "smart wallets". The complexity and details of the approval process are left to the discretion of implementers.

Only the owner or keys with the `EXECUTE` permission can execute transactions.

#### Execute
Passes an execution instruction to the Key Manager.
**SHOULD** require approval to be called with one or more keys to approve this execution.

Returns `executionId`: SHOULD be sent to the `approve` function, to approve or reject this execution.

**Triggers Event**: `ExecutionRequested`

**Triggers on direct execution Event**: `Executed`

```solidity
function execute(address _to, uint256 _value, bytes _data) public hasPermission(EXECUTE) returns (uint256 executionId);
```

#### Approve
Approves an execution or claim addition.
This **SHOULD** require the approval of the key with the appropriate permissions.

Only the owner or keys with the `APPROVE` permission can approve executions or claims.

**Triggers Event**: `Approved`

**Triggers on successful execution Event**: `Executed`

```solidity
function approve(uint256 _id, bool _approve) public hasPermission(APPROVE) returns (bool success);
```

### Events

Events are very important because most Identity management applications will need these to display appropriate information about the identity state.

#### `KeyAdded`

MUST be triggered when `addKey` was successfully called.

```solidity
event KeyAdded(address indexed keyAddress);
```

#### `KeyRemoved`

MUST be triggered when `removeKey` was successfully called.

```solidity
event KeyRemoved(address indexed keyAddress);
```

#### `PermissionAssigned`

MUST be triggered when `assignPermission` was successfully called.

```solidity
event PermissionAssigned(address indexed keyAddress, bytes32 permission);
```

#### `PermissionRevoked`

MUST be triggered when `revokePermission` was successfully called.

```solidity
event PermissionRevoked(address indexed keyAddress, bytes32 permission);
```

#### `ExecutionRequested`

This event means an execution request was created and is awaiting approval (note: if the execution was immediately approved and then performed, the ExecutionRequested would be followed by an Executed event in the same transaction).

MUST be triggered when `execute` was successfully called.

```solidity
event ExecutionRequested(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
```

#### `Executed`

MUST be triggered when `approve` was called and the execution was successfully approved.

```solidity
event Executed(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
```

#### `Approved`

MUST be triggered when `approve` was successfully called.

```solidity
event Approved(uint256 indexed executionId, bool approved);
```

## Rationale

The rationale behind this approach is to provide a flexible identity management system that works for both simple and complex use cases. By implementing a minimal KMS with full permissions by default and allowing custom permissions as an optional feature, ONCHAINID accommodates a wide range of applications, from lightweight identity management to enterprise-level requirements.

Custom permissions provide flexibility for integrating with external contracts that use ONCHAINID for access control. For example, an external `PizzaOrdering` contract could define a custom permission like `ORDER_PIZZA` and use ONCHAINID to determine if a key has the required permission before allowing an order to be placed. This allows for decentralized access control that can extend beyond the core functions of ONCHAINID itself.

## Backwards Compatibility

There are no known standards using the previous KeyHolder and ClaimHolder proposals. Most methods used are inspired by specifications of KeyHolder and ClaimHolder introduced by Fabian Vogelsteller (@frozeman), hence applications leveraging claims and keys functionalities should still be compatible.

## Security Considerations

### Privacy

Because claims may be related to personal information, developers and especially claim issuers are expected to consider the privacy implications of the claims they issue. Especially:
- The content of the claim `data` property is to be considered public and should never contain sensitive information, even encrypted. For such information, it is recommended to use an integrity hash of information stored offchain combined with a random string and to include the hash in the claim data. Digest algorithm choice is left to claim issuers and they may differ from one to another.
- The existence of a claim with a given topic could reveal information about the identity. Claim issuers should not issue claims that are too specific and should avoid claims when not necessary.

When using an Identity, identity owners (and services managing identities) should keep in mind that:
- Identity information should, as much as possible, be kept off-chain. On-chain claims should only be added to the identity when they are necessary (usually to interact with permissioned smart contracts).

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
