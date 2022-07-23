# Farcaster Protocol

## Contents

1. [Introduction](#1-introduction)
2. [Concepts](#2-concepts)
   1. [Accounts](#21-accounts)
   2. [Signed Messages](#22-signed-messages)
   3. [Applications](#23-applications)
   4. [Hubs](#24-hubs)
3. [Identity Registry](#3-identity-registry)
   1. [Account Registry](#31-account-registry)
   1. [Namespace Contract](#32-namespace-registry)
   1. [Recovery](#33-recovery)
4. [Replication](#4-replication)
   1. [Casts](#41-casts)
   2. [Actions](#42-actions)
   3. [Verifications](#43-verifications)
   4. [Metadata](#44-metadata)
   5. [Signer Authorizations](#45-signer-authorizations)
   6. [Root Signer Revocations](#46-root-signer-revocations)
   7. [Sharding](#47-sharding)
5. [Peering](#5-peering)
6. [Upgradeability](#6-upgradeability)
   1. [Minor Upgrades](#61-minor-upgrades)
   2. [Major Upgrades](#62-major-upgrades)
7. [Security Considerations](#7-security-considerations)
   1. [Signer Compromise](#71-signer-compromise)
   2. [Eclipse Attacks](#72-eclipse-attacks)
   3. [Flooding Attacks](#73-flooding-attacks)
   4. [DDOS Attacks](#74-ddos-attacks)
   5. [Replay Attacks](#75-replay-attacks)
8. [URIs](#8-uris)
9. [Governance](#9-governance)

# 1. Introduction

Social networks have become an essential part of our lives over the last decade. Many began their journey as open platforms, courting developers to build on their APIs. These developers created new clients, discovered new UI paradigms, and even launched multi-billion dollar businesses that brought in many users. However, networks have turned away from developers over the last few years. They have restricted APIs, implemented arbitrary review processes, and removed access with little notice or recourse.

Farcaster is a [sufficiently decentralized](https://www.varunsrinivasan.com/2022/01/11/sufficient-decentralization-for-social-networks) protocol that empowers developers to build novel social networks. We define a sufficiently decentralized network as one where **two users who want to communicate are always able to, even if the network wants to prevent it**. Users on such networks must have complete control over their identity (usernames), data (messages), and social graph (relationships to others). If a third party controls any of these, they can prevent two users from communicating. Developers must also be free to build applications and have unrestricted access to the network, and users must be free to switch between them. If there was only one app to connect to the network, it could prevent two users from communicating.

# 2. Concepts

Farcaster achieves sufficient decentralization through a hybrid architecture with on-chain and off-chain components.

User identities are stored on-chain in an Ethereum smart contract to leverage Ethereum's robust security, composability, and consistency guarantees. An Ethereum address controls this on-chain identity and can be used to sign off-chain messages on its behalf.

User data is then cryptographically signed by the identity and stored off-chain on user-controlled servers called Farcaster Hubs. Data is not stored on-chain because it would be prohibitively expensive and slow to settle on most L1 and L2 networks.

<!-- Diagram covering the major architectural concepts  -->

## 2.1 Accounts

A Farcaster account is similar to an account on pseudonymous social networks like Twitter or Reddit. Individuals can operate several accounts simultaneously, like a real-name account and a company account.

An Ethereum address can mint a new account from the _AccountRegistry_, which issues an account number. This address is known as the `custody address`, and it can sign messages on behalf of the account. Accounts can be enriched by adding a profile picture, display name, biography, and verified usernames like alice.eth, which are set off-chain with signed messages.

## 2.2 Signed Messages

Signed Messages are **tamper-proof** and **self-authenticating** objects that are signed by an account.

```ts
type SignedMessage = {
  message: {
    body: any;
    account: number;
    timestamp: number;
  };
  envelope: {
    hash: string;
    hashType: 'BLAKE2b';
    signature: string;
    signatureType: 'ed25519' | 'ecdsa-secp256k1';
    signerPubKey: string;
  };
};
```

A Signed Message has a **message** property that contains the payload. The payload is then serialized, hashed, and signed by a valid keypair, like the custody address. The **envelope** contains the hash, signature, and the public key of the signing keypair, which any recipient can use to validate that the account signed the message.

The message must be serialized with [RFC-8785](https://datatracker.ietf.org/doc/html/rfc8785), hashed with [BLAKE2b](https://www.rfc-editor.org/rfc/rfc7693.txt) and signed with an Ed25519 signature scheme. Each message must also contain an account number to look up the custody address on-chain and a timestamp for ordering.

## 2.3 Applications

An _application_ is a program that people use to interact with the Farcaster network. Users can choose the type of application that best suits their needs and switch between them at any time.

A simple application might consist of a standalone desktop or mobile client that talks directly to a Farcaster Hub. It can publish new messages and view messages published by other accounts. Such applications are **self-hosted** and must be instantiated with the custody address or [valid signing key](#45-signer-authorizations).

A more sophisticated application might add a proxy backend server that indexes data from Hubs. Indexing allows servers to implement features like search, algorithmic feeds, and spam detection that are difficult or expensive to perform on the Hub. Such applications can be **self-hosted** by storing keys on the client; **delegated** by asking users for a [delegate signing key](#45-signer-authorizations); or **hosted** by managing all keys including the custody address.

## 2.4 Hubs

A Hub is an always-on server that validates, stores, and replicates Signed Messages.

Users must upload messages they create to a primary Hub and publish its URL on-chain using the AccountRegistry. Their followers can use this Hub to find and download their messages. Users can run a Hub themselves or use a third-party Hub service. They are incentivized to ensure that it works correctly, or their followers will not receive their messages.

Users can also configure their primary Hub to replicate data from other Hubs. If Alice follows Bob and Charlie, who use separate Hubs, she can configure her Hub to download messages from theirs. When her client comes online, it can make a single request to her Hub and fetch Bob and Charlie's messages.

Hubs maintain a connection to the Account Registry to validate every Signed Message they receive. A malicious Hub that served a forged message would be detected because the message authentication would fail. This property of Signed Messages lets us safely receive messages signed by _any_ user from _any_ Hub. If Bob has a copy of Charlie's messages, Alice's server can download them and save a round trip to Charlie's Hub. Hubs can fetch data from nearby peers using a gossip-based pubsub protocol [^gossip-sub] instead of making a round trip to each user's primary Hub.

Conceptually, Hubs form an **L2 network for storing social data**, though the network has different properties from blockchain-based L2s. Its consensus model has weaker consistency guarantees but stronger scalability guarantees because the network data is **shardable** down to the account level.

# 3. Identity Registry

The identity registry ensures secure and decentralized ownership of user accounts. Users must be able to set up a new account without any third-party approval. It should also be reasonably quick and cheap, which we define as being able to sign up in under a minute and for less than $10. The system should also ensure that most accounts are trustworthy and operated by honest, active users.

Farcaster's Identity Registry comprises two smart contracts - an **AccountRegistry**, which issues new accounts, and a **NamespaceRegistry** which issues usernames for accounts.

## 3.1 Account Registry

An account is created by calling `register` on the AccountRegistry from a user-controlled Ethereum address. The contract issues a new account number and links it to this address. Accounts can be transferred between addresses, though the AccountRegistry ensures that an address owns no more than one account at a time.

Account numbers start at 0 and are incremented by one every time a new account is registered, which is a gas-efficient way to ensure unique account numbers. Numbers are represented internally with a uint256 and have a practically infinite supply since they can be incremented to ~ 10^77.

## 3.2 Namespace Registry

A Farcaster Name like `@alice` can be minted from the NamespaceRegistry and used on the protocol.

When minting a name, the contract checks that the name is unique and contains at most 16 alphanumeric characters or dashes `^[a-zA-Z0-9-]{1,16}$`. The single dash (-) username is reserved for a system account and will not be claimable. The mint happens over a two-phase commit reveal to prevent front-running registrations.

Farcaster Names are ERC-721 tokens that are fully composable with the NFT ecosystem. While users can use ENS names with the protocol, Farcaster Names have some properties that make them more practical. Names are cheaper to mint and are [recoverable](#recovery) if the address holding them is lost. They are also less vulnerable to [homoglyph attacks](https://en.wikipedia.org/wiki/IDN_homograph_attack) and more straightforward to display because of the restricted length and character set.

The Namespace Registry charges a yearly fee of 0.01 ETH for owning a name, which is charged yearly on Farcaster Day (August 1st). Users who do not renew their usernames by this day have a 30-day grace period after which their ownership over the name expires. During mint, the fee is pro-rated for the year ending August 1st. Fees are collected by the Farcaster Treasury and are used to support protocol development.

Names can be minted freely but are subject to two policies enforced by governance:

1. A _prior ownership_ claim can be made by a user who owns a name on at least two major social media platforms if the current owner does not own the name on any major social media platforms.

2. An _impersonation claim_ can be made if someone is impersonating or misleading users. It will be investigated by a group of moderators elected by the community through a governance vote.

Names that expire or are subject to a claim are moved back into the Farcaster Treasury. The treasury can choose to transfer ownership to someone with ownership claims or auction them to the highest bidder. For impersonation claims, any previously paid fees are forfeit.

## 3.3 Recovery

Names and Addresses are recoverable if the user loses the keys to the address holding them. Both contracts implement a time-delayed recovery system that allows a **recovery address** to request a transfer to a new address. If the custody address does not cancel the transfer within three days, the recovery address can complete the transfer.

Users can set the recovery address to another address in their wallet, a multi-sig shared with friends, or a third-party recovery service. Users can also change the recovery address at any time. Ownership remains decentralized because the recovery address cannot make a transfer that the custody address does not approve.

Transferring the asset to a new custody address must unset the recovery address. Otherwise, users may purchase a name on OpenSea only to have the previous owner claim it back stealthily with their recovery address.

# 4. Replication

_Replication_ is the process by which Hubs accept new messages and determine a user's state.

Users send [messages](#22-signed-messages) to a Hub for every action they take. If a user likes a URL, unlikes it, and likes it again, that creates three messages. A Hub that receives all messages will determine the current state of the URL as _liked by the user_. The Hub can discard the first two messages to save space since they are no longer needed. Hubs may condense messages like this using a merge operation, which avoids client-level disagreement and saves space. Messages may have different rules for their merge operations. For example, two likes on the same cast by a user can be condensed into one, while two replies cannot.

A Hub may fail to receive some messages from a user and end up in a partial state. For instance, it may just get the first like and the unlike which sets the current state to _not liked by the user_. The merge operation should allow such partial merges to move the state forward and reach consistency when the missing messages are re-broadcast. In other words, the merge should ensure [strong eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency#Strong_eventual_consistency).

Hubs achieve this by implementing a [CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) Set for each message type which encodes specific validation and merge rules. This property makes Hubs highly available since they can go offline at any time and always get back into sync. Formally, our CRDT sets are anonymous Δ-state CRDTs[^delta-state], and each message is a join-irreducible update on the set.

<!-- Diagram of all user data types -->

#### Message Ordering

Sets can order Signed Messages by their timestamp to resolve merge conflicts with a last write wins strategy. However, they cannot guarantee perfect ordering since timestamps are vulnerable to [clock skew](https://en.wikipedia.org/wiki/Clock_skew), [clock drift](https://en.wikipedia.org/wiki/Clock_drift), spoofing from malicious users and may collide for valid reasons. Applications can use [hybrid clocks](https://martinfowler.com/articles/patterns-of-distributed-systems/hybrid-clock.html) to generate perfectly ordered timestamps that do not collide, but we cannot enforce their usage.

Instead, we define an ordering system for messages that ensures total ordering by using timestamps to determine initial order and hashes to break conflicts. Total ordering is guaranteed because two messages cannot have the same hash unless they are the same message. Two messages `a` and `b` can be compared with this algorithm:

- If `a.timestamp > b.timestamp`, `a` is greater.
- If `a.timestamp < b.timestamp`, `b` is greater
- If `a.timestamp == b.timestamp`
  - If `a.hash > b.hash`, `a` is greater
  - If `a.hash < b.hash`, `b` is greater
  - If `a.hash = b.hash`, `a == b`

Timestamps are compared as numbers, and hashes are compared as strings. Since string comparison can vary across implementations, we must be precise in our comparison algorithm. We say that two hashes `x` and `y` can be compared by comparing each pair of characters, starting from the first:

- If all character pairs are equal and `x` and `y` terminate, then `x == y`
- If all character pairs are equal and `x` terminates first, then `y > x`
- If a differing character pair `xC, yC` is encountered, then `y > x` if `ASCII(yC) > ASCII(xC)`

#### Message Validation

All messages must pass the following validations in addition to specific validations for the message type:

1. `message.timestamp` is not more than 1 hour ahead of system time.
2. `message.account` must be a known account number in the AccountRegistry.
3. `signerPubKey` should be a valid [Root Signer or Delegate Signer](#45-signer-authorizations) for `message.account`
4. `hashFn(serializeFn(message))` must match `envelope.hash`, where hashFn is a Blake2B function and serializeFn performs JSON canonicalization.
5. `EdDSA_signature_verify(envelope.hash, envelope.signerPubKey, envelope.signature)` should pass.

## 4.1 Casts

A _Cast_ is a public message created by a user that is displayed on their profile. It can contain text and links to media, on-chain activity or other casts. Users are allowed to delete casts at any time. Casts come in several different flavors and the protocol can be extended to support many more types in the future. Each type has its own replication mechanisms which are defined below.

### 4.1.1 Short Text Casts

A _Short Text Cast_ is a 280 character public message created by an account. It can stand on its own or as a reply if `parentUri` points to another cast, on-chain item or URL. When deleted, a Short Text Cast should be soft-deleted and its content removed, but its replies are still considered valid and can be displayed to the user.

```ts
type CastShortTextBody = {
  embed: Embed;
  text: string;
  schema: 'farcaster.xyz/schemas/v1/cast-short-text';
  parentUri?: URI;
};
```

Short Text Casts can be represented as a series of trees, where each root is a Cast or URI, and each node is a Cast. This is a useful property for threading, since there is a determinstic order to how messages should be displayed in the UI during a back and forth conversation. It is impossible to break such a tree by introducing a cycle because of the requirement that every node must be hashed and signed before a child can reference it. If the parent or child attempts to change their `parentUri` after finalization, their hash changes making them a distinct node in the tree.

<!-- Diagram of a Set of Short Text Casts -->

#### Message Validation

1. `schema` must be known.
2. `text` must contain <= 280 valid unicode characters,
3. `parentUri` must be a valid Farcaster Cast URI and must not reference this message.

#### Set Construction

Short Text Casts for a user are stored in a two-phase set CRDT[^two-phase-set], wwhich contains an **add-set** that stores additions and a **rem-set** which stores [removes](#4). An addition is performed with a `CastShortTextBody` message, while a remove is performed with a `CastRemove`.

When an addition message `c` is received:

1. If there exists `r` in the rem-set such that `r.targetCastHash` equals `c.hash`, discard `c`
2. Otherwise, add `c` into the add-set.

When a remove message `r` is received:

1. If there is an `a` in the add-set where `a.hash` equals `d.targetCastHash`, delete it.
2. If there is an `r` in the rem-set where `r.targetCastHash` equals `d.targetCastHash`
   - If `r > d`, discard `d`
   - If `r < d`, delete `r` and add `d` into the rem-set
3. Otherwise, add `d` to the rem-set.

### 4.1.2 Recasts

A _Recast_ expresses an intent to share another cast. An account can recast any cast, including its own, but may not recast the same cast multiple times.

```ts
type CastRecast = {
  targetCastUri: URI;
  schema: 'farcaster.xyz/schemas/v1/cast-recast';
};
```

#### Message Validation

1. `schema` must be known.
2. `targetCastUri` must be a valid Farcaster Cast URI and must not reference this message.

#### Set Construction

Recasts can be stored using a two-phase set just like Short Text Casts. The remove operation is identical, while the addition operation is performed with `CastRecast`, and has one additional rule:

1. If there is a CastRecast `a` in the add **add-set** with the same `targetCastUri` and `account` as the incoming CastRecast `rc`:
   - If `a > rc`, discard `d`
   - If `a < rc`, delete `a` and add `rc` into the rem-set

### 4.1.3 Remove Messages

A remove instructs the set to remove a previously created cast. It is a type of soft delete where the content of the message is removed but its hash is retained forever. A user who has a copy of the remove message can prove that the message was posted at some point by the author.

```ts
type CastRemove = {
  targetCastHash: string;
  schema: 'farcaster.xyz/schemas/v1/cast-remove';
};
```

The remove message must contain the hash of the cast being removed and omit all the other properties. The set can then remove the cast and its contents from the network, which is desirable from a user perspective. The set ensures that the remove message does not reference itself, and that it is only applied as a remove operation if it has a timestamp higher than that of the message it references. The rules for validation are described below, but the rules for merging are specified in each set that implements the remove operation.

<!-- Diagram showing the remove operations -->

**Validation**

1. `schema` must be known.
2. `targetCastHash` must not equal the `hash` of this message.
3. `timestamp` must be <= system clock + 1 hour
4. `account` must be a known account number in the AccountRegistry

### 4.1.4 Embeds

An _Embed_ is a data structure that can attach Farcaster URI's to a Cast, which clients can choose to preview when displaying the message.

```ts
type Embed = {
  items: URI[];
};
```

**Validation**

1. Every item in `items` must be a valid Farcaster URI

## 4.2 Actions

An action is a public operation performed by the user on a target, which can be another user, cast or on-chain activity. Two types of actions are supported today: **likes** and **follows**. The protocol can be extended to support new actions easily. Users can undo and redo actions by toggling the `active` property on the message. Conceptually, each action is an edge in a social graph.

```ts
type Action = {
  message: {
    body: {
      active: boolean;
      type: 'like' | 'follow';
      targetUri: FarcasterURI;
      schema: 'farcaster.xyz/schemas/v1/action';
    };
    account: number;
    timestamp: number;
  };
};
```

#### Message Validation

1. `schema` must be known.
2. `active`, `type` must be present and match types.
3. `targetUri` must be a valid FarcasterURI and must not reference this message.

#### Set Construction

Actions are managed with an [LWW-Element-Set CRDT](<https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#LWW-Element-Set_(Last-Write-Wins-Element-Set)>) which guarantees strong eventual consistency. Conceptually, there is a single **set** that stores all messages and conflicts are resolved by timestamp and lexicographical hash order. An addition is performed by constructing an `Action` message `a` where `active` is true, while a remove is performed by setting `active` to false. In both cases, the logic for merging the message into the set is:

1. If there is an action `x` in the set with the same values for `type`, `targetUri` and `account` as the incoming action `y`:
   - If `x > y`, discard `y`
   - If `x < y`, delete `x` and add `y` into the set
2. Otherwise, add `y` into the set.

## 4.3 Verifications

Verifications are bi-directional proofs of ownership between Farcaster accounts and external entities. Verifications can be used to prove ownership of Ethereum addresses, specific NFTs, other social media accounts, or even domains.

Verifications have three core concepts:

1. A claim that includes a reference to a Farcaster account and the external entity. The claim can be hashed to create a unique identifier for each claim.
2. A directional proof from the external entity that is authorized to make the claim, showing intent to connect it to the Farcaster account.
3. A directional proof from the Farcaster account accepting the request to associate the claim with the Farcaster account.

Verification claims are structured as `VerificationClaim` objects:

```ts
type VerificationClaim = {
  externalAddressUri: URI;
  account: number;
};
```

Verifications are added via `VerificationAdd` messages and removed via `VerificationRemove` messages. Here are the types for the add and remove message bodies:

```ts
type VerificationAddBody = {
  externalAddressUri: string;
  claimHash: string;
  externalSignature: string;
  externalSignatureType: 'eip-191-0x45'; // Will support other types over time
  schema: 'farcaster.xyz/schemas/v1/verification-add';
};

type VerificationRemoveBody = {
  claimHash: string;
  schema: 'farcaster.xyz/schemas/v1/verification-remove';
};
```

Notice how `VerificationAddBody` does not include the full claim object, because the claim can be reconstructed from the `externalAddressUri` and `account` attributes of the add message.

#### Message Validation

The `envelope` of `VerificationAdd` and `VerificationRemove` messages is validated the [same as other message types](#message-validation).

Here are rules for `VerificationAdd` messages:

1. `schema` must be known
2. `claimHash` must be present and match `hashFn(claim)`
3. `externalSignatureType` must be known
4. `externalSignature` must be valid according to custom rules depending on `externalSignatureType`, which are defined in subsections below

Here are rules for `VerificationRemove` messages:

1. `schema` must be known
2. `claimHash` must be present

#### Set Construction

Verifications are stored in the Verifications Set, which is a modified [Last-Write-Wins-Element Set](<https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#LWW-Element-Set_(Last-Write-Wins-Element-Set)>). Valid and unremoved verifications are stored as `VerificationAdd` messages in the `adds` set. Removed verifications are stored as `VerificationRemove` messages in the `removes` set. Messages are indexed by `claimHash` and a particular claim cannot exist in both sets simultaneously.

Verifications can be re-added once they've been removed, and conflicts are resolved using timestamp. Verification information like the external entity address is effectively hidden once a verification is removed, because the `VerificationRemove` message only contains the claim hash.

When a verification add message `a` is received:

1. If `a` exists in the `removes` set
   1. If existing remove message is more recent, discard `a`
   2. Otherwise, move `a` to `adds` set
2. If `a` exists in the `adds` set
   1. If existing add message is more recent, discard `a`
   2. Otherwise, overwrite `a` in the `adds` set with the new message
3. If `a` does not exist in either set
   1. Add `a` to `adds` set

When a verification remove message `b` is received:

1. If `b` exists in the `removes` set
   1. If existing remove message is more recent, discard `b`
   2. Otherwise, overwrite `b` in the `removes` set with the new message
2. If `b` exists in the `adds` set
   1. If existing remove message is more recent, discard `b`
   2. Otherwise, move `b` from `adds` to `removes`
3. If `b` does not exist in either set
   1. Add `b` to `removes` set

### 4.3.1 Ethereum address verifications

The first type of verification supported is a self-authenticating proof of ownership of an Ethereum address. The `externalSignatureType` attribute for Ethereum address verifications is `eip-191-0x45`.

The bi-directional proof is created in three steps:

1. Create a `VerificationClaim` object containing the Farcaster account and Ethereum address and hash the claim to create a unique claim hash.
2. Have the Ethereum address sign the claim hash according to [EIP 191 version 0x45](https://eips.ethereum.org/EIPS/eip-191) (i.e. Ethereum's `personal_sign`).
3. Create a `VerificationAdd` message containing the Ethereum address, claim hash, and signature and have the Farcaster account sign it like other messages.

The `externalSignature` for an Ethereum address verification must be a valid [EIP 191 version 0x45](https://eips.ethereum.org/EIPS/eip-191) signature of the `claimHash` by `externalAddressUri`.

## 4.4 Metadata

_This section is still under development and will cover a CRDT for allowing arbitrary metdata to be added to a user's account like a display name or profile picture._

## 4.5 Signer Authorizations

_This section is still under development._

A _Signer Authorization_ is a message that authorizes a new key pair to generate signatures for a Farcaster account.

When an account is minted, only the custody address can sign messages on its behalf. Users might not want to load this keypair into every device since it increases the risk of account compromise. The custody address, also known as the _Root Signer_, can authorize other keypairs known as _Delegate Signers_. Unlike Root Signers, a Delegate Signer is only allowed to publish off-chain messages and cannot perform any on-chain actions.

```ts
type SignerAuthorizationMessage = {
  account: number;
  active: boolean;
  authorizedPublicKey: string;
  schema: 'farcaster.xyz/schemas/v1/signer';
};
```

Root Signers generate ECDSA signatures on the secp256k1 curve and can only publish Signer Authorization messages. All other types of messages must be signed by Delegate Signers, which creates EdDSA signatures on Curve25519[^ed25519]. Delegate Signers can be used to authorize new devices or even third-party services to sign messages for an account. If a Delegate Signer is compromised, it can be revoked by itself, an ancestor in its chain of trust, or any Root Signer. When a Signer is revoked, Hubs discard all of its signed messages because there is no way to tell the user's messages from the attackers.

Users might also transfer an account to a new custody address due to key recovery or changing wallets. It is usually desirable to preserve history and therefore both custody addresses become valid Root Signers. The set of valid signers for an account form a series of distinct trees. Each tree's root is a historical custody address, and the leaves are delegate signers.

<!-- Diagram of Signer Tree -->

The Signer Set is a modified two-phase set with remove-wins and last-write-wins semantics. New messages are added to the set if signed by a valid delegate or root signer. A remove message is accepted if signed by itself or by an ancestor. A Signer can never be re-added once removed, and all of its descendant children and messages are discarded.

A Set conflict can occur if two valid Signers separately authorize the same Delegate Signer, which breaks the tree data structure. If this occurs, the Set retains the message with the highest timestamp and lexicographical hash, in that order.

## 4.6 Root Signer Revocations

_This section is still under development._

A Root Signer Revocation is a special message that is used to remove previous custody addresses from the list of valid signers. This is useful if you believe that a previous address may have become compromised or if you are changing ownership of an account.

A revocation must include the blockchash of a specific Ethereum block. It must be signed by a custody address that owned it at the end of that block or afterwards. When received, the custody address at the end of the block specified is considered the first valid root signer. All previous custody addresses and delegate signers issued by them are invalidated.

```ts
type RootRevocationBody = {
  blockHash: string;
  schema: 'farcaster.xyz/schemas/v1/root-revocation';
};
```

## 4.7 Sharding

Hubs can replicate data for specific accounts only, which is a useful property for scaling the network. If Farcaster grows large enough that a single server cannot support a Hub that replicates the entire network, the workload can be sharded across multiple Hubs. Hub operators can also avoid syncing data for users who are behaving maliciously or not relevant to the operator.

Selective replication only provides a partial view of the network. If a Hub is syncing Alice's data it will become aware that she replied and liked one of Bob's posts. However, it will not know the contents of Bob's post, or the fact that Bob liked her reply and then proceeded to reply to it. An application that seeks to provide accurate like counts and serve up all the replies to a message should replicate as many users as possible.

# 5. Peering

This section is still under development and covers the protocols by which Hubs discover and exchange signed messages with their peers.

# 6. Upgradeability

Farcaster is intended to be a long-lived protocol and built on the idea of [stability without stagnation](https://doc.rust-lang.org/1.30.0/book/second-edition/appendix-07-nightly-rust.html). Upgrades are designed to be regular, painless and bring improvements for users and operators. New versions of the Hub will be released every 6 weeks. Upgrading should require just a few minutes of downtime for the operator in most cases. Hubs come with a "time bomb" which forces them to shut down 2 weeks after a new release is published.

## 6.1 Minor Upgrades

A minor upgrade is a backwards compatible change to the Farcaster consensus protocol. The old and new versions of the protocol can run safely side by side and reach consensus on the network. Some examples of minor upgrades including adding a new type of message, removing an existing message type or modifying message schemas to add new, optional properties.

## 6.2 Major Upgrades

A major upgrade is a backwards incompatible change to the Farcaster consensus protocol. If the new protocol and the old protocol were run side by side, they might end up in a divergent state. Some example of upgrades that may require major changes are changes to the merge logic in sets or adding new required properties to schemas.

Backwards incompatible changes to an existing schema are possible, but discouraged since they increase protocol complexity. The Hubs must assume that old data and new data can live side-by-side and will have to have special consensus rules to handle both.

Hub releases that include backwards incompatible changes must implement both consensus algorithms. They are programmed to run the old consensus model until a specific Ethereum block is mined. This block is chosen such that it will occur after the release time bomb goes off, which gives us high confidence that all nodes are running the new version and switch to the new algorithm at once.

# 7. Security Considerations

## 7.1 Signer Compromise

A malicious attacker can use a compromised delegate signer to impersonate the user by signing messages. As long as the user has control over a parent signer or the root signer, they can invalidate the signare and issue a new one. Unfortunately, this means that all messages signed by that signer will be lost since we cannot tell which ones were signed by the attacker. Clients can mitigate this by constantly rolling keys after every few thousand messages, which limits the scope of how many messages will be lost when a signer is reset.

## 7.2 Eclipse Attacks

A malicious user could spin up several Hubs which pretend that a target user has published zero messages. Peers might assume this to be true, effectively blocking the target from the network. Hubs can maintain an internal score for each peer based on data availability. Periodically, they lookup the location of the user's source Hub which is published in the AccountRegistry and sync the latest messages. If their peers do not have an up-to-date copy of this, their scores are lowered until eventually the peer is dropped and a new one is selected.

## 7.3 Flooding Attacks

A flooding attack is when a malicious user keeps acquiring new accounts and broadcasting thousands of messages. Hubs may attempt to sync this data bloating their on-disk storage and causing network congestion which leads to stale data for legitimate user accounts. An account scoring system can help alleviate this by prioritizing trustworthy accounts and temporarily or permanently banning misbehaving accounts. Each Hub tracks its own score for an account which starts at zero. The score increases if the account has a valid username and a history of behaving well. It decreases when malicious behavior like flooding is observed.

## 7.4 DDOS Attacks

A DDOS attack is when a malicious Hub spams a target Hub with queries that are expensive to run causing it to stop responding to legitimate requests and syncing with other Hubs. A simple mitigation strategy is to implement IP-based rate limiting that rejects query requests from Hubs when they exceed a threshold. More sophisticated DDOS attacks might require aggressive mitigations like whitelisting a set of known peers or relying on infrastructure-level DDOS protection services offered by cloud vendors.

## 7.5 Replay Attacks

A replay attack is when a malicious actor stores a copy of a user's message and is able to replay it to cause an action that the user did not intend. This attack is possible if a Signer is removed, since it may delete a message that causally removed a prior message from the state. Assume Alice added a "hello world" cast with Signer A and then deleted it with Signer B and then proceeded to delete Signer B. At this point, the cast is no longer present on the Hub but neither is the delete message. A malicious user with a copy of the "hello world" cast could replay it causing it to be accepted as valid. Fortunately, the attack is restricted to replaying previously valid messages and Alice can issue a remove message from a currently valid signer to nullify it.

# 8. URIs

This section is still under development and will cover a schema for URIs supported by Farcaster Message types.

# 9. Governance

Farcaster is a decentralized protocol that is not controlled by a single individual, and governance is the process of making protocol changes in a decentralized way. The process is kept lightweight during beta to encourage community contributions and rapid development cycles.

Anyone can propose a change by opening up a [new discussion topic](https://github.com/farcasterxyz/protocol/discussions) in the protocol repository. The Farcaster team will provide feedback and may ask for more details. Once all feedback has been provided the Farcaster team will decide whether to include the change in the roadmap or whether to reject it. Once approved, an issue is created and the specification changes are merged into this repository.

## Hub Changes

Changes that involve off-chain systems must be implemented and deployed in the Hubs. The Farcaster team will work closely with Hub developers and operators to pick a release date that will ensure a smooth transition. The Farcaster team is also responsible for ensuring that there is strong alignment around implementing the changes.

Developers and operators can veto a change if they disagree with it, but at some cost to themselves and the network. An operator may choose not to upgrade their Hub version and a developer can choose not to release the change. This will cause fragmentation and users on such Hubs may not be visible to the rest of the network. It is desirable for developers and operators to have this power to ensure decentralization of the network, but ideally they would never need to exercise it.

## Contract Changes

Changes that involve on-chain systems must be implemented by deploying a new contract or upgrading an existing one. The Farcaster team will implement these changes and ensure that they are thoroughly audited. Contracts will be controlled by a multi-sig whose ownership is split between members of the Farcaster team during beta. Over time, control over making contract changes will be decentralized to other parties who have a vested interest in ensuring the success of the network.

[^gossip-sub]: Dimitris Vyzovitis, Yusef Napora, Dirk McCormick, David Dias, Yiannis Psaras: “GossipSub: Attack-Resilient Message Propagation in the Filecoin and ETH2.0 Networks”, 2020; [http://arxiv.org/abs/2007.02754 arXiv:2007.02754].
[^delta-state]: van der Linde, A., Leitão, J., & Preguiça, N. (2016). Δ-CRDTs: Making δ-CRDTs delta-based. Proceedings of the 2nd Workshop on the Principles and Practice of Consistency for Distributed Data. https://doi.org/10.1145/2911151.2911163p
[^two-phase-set]: Shapiro, Marc; Preguiça, Nuno; Baquero, Carlos; Zawirski, Marek (2011). "A Comprehensive Study of Convergent and Commutative Replicated Data Types". Rr-7506.
[^ed25519]: Bernstein, D.J., Duif, N., Lange, T. et al. High-speed high-security signatures. J Cryptogr Eng 2, 77–89 (2012). https://doi.org/10.1007/s13389-012-0027-1
