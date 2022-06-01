# Farcaster Protocol


## Contents
1. [Introduction](#1-introduction)
2. [Concepts](#2-concepts)
    1. [Accounts](#21-accounts)
    2. [Signed Messages](#22-signed-messages)
    3. [Applications](#23-applications)
    4. [Hubs](#24-hubs)
3. [Identity System](#3-identity-system)
    1. [Account Contract](#31-account-contract)
    1. [Namespace Contract](#32-namespace-contract)
    3. [Recovery](#33-recovery)
4. [Replication](#4replication)
    1. [Signers](#41-signers)
    2. [Casts](#42-casts)
    3. [Actions](#43-actions)
    4. [Verifications](#44-verifications)
    5. [Metadata](#45-metadata)
    6. [Root Revocations](#46-root-revocations)
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

Social networks have become an essential part of our lives over the last decade. Many of them began their journey as open platforms, courting developers to build on their APIs. These developers created new clients, discovered new UI paradigms, and even launched multi-billion dollar businesses that brought in many users. But over the last few years networks have turned away from developers. They've restricted APIs, implemented arbitrary review processes and have removed access with little notice or recourse. 

Farcaster is a [sufficiently decentralized](https://www.varunsrinivasan.com/2022/01/11/sufficient-decentralization-for-social-networks) protocol that empowers developers to build novel social networks. We define a sufficiently decentralized network as one where **two users who want to communicate are always able to, even if the network wants to prevent it**. Users on such networks must have full control over their identity (usernames), data (messages) and social graph (relationships to others). If a third party controls any of these, they can prevent two users from communicating. Developers must also be free to build applications and have unrestricted access to the network and users must be free to switch between them. If there was only one app to connect to the network, it could prevent two users from communicating.  

# 2. Concepts

Farcaster achieves sufficient decentralization through a hybrid architecture that has on-chain and off-chain components. 

User identities are kept on-chain because they are valuable assets, and we want to leverage the security, composability and strong consistency guarantees of the Ethereum blockchain. On-chain identities are controlled by an Ethereum address, which is a public private key pair that can also sign off-chain messages.  

User data cannot be practically stored on the Ethereum blockchain or any other L1 because of the strong privacy and scalability requirements. Instead users store their data off-chain on a server under their control known as a Farcaster Hub. All data must be cryptographically signed by the user's identity address before being published to the Hub. 

<!-- Diagram covering the major architectural concepts  -->

## 2.1 Accounts

An Farcaster account is similar to an account on pseudonymous social networks like Twitter or Reddit. An individual can operate several accounts as the same time, like a real-name account, a pseudonymous account and a company account.

An account is created by minting a new account number from the Account contract on-chain. Each account number is like a primary key in a database, and points to a user-controlled address known as the `custody address`. Once an account is minted, the custody address can sign messages on behalf of the account. Accounts can be enriched by adding a profile picture, display name, biography and verified usernames like `alice.eth`, which are all done off-chain with signed messages. 

## 2.2 Signed Messages

Signed Messages are cryptographically signed messages from an account that are **tamper-proof** and **self-authenticating**. 

A Signed Message has a **message** object, which contains the data that will be signed. The message is then serialized, and then hashed and signed with the custody address. The hash, signature and signer public key are used to construct the **envelope**, which proves that the message is authentic and has not been modified. Any observer can verify the message by checking that: 

1. The account number is owned by an address whose public key matches the public key in the envelope
2. The serialized hash of message matches the hash in the envelope
3. The signature in the envelope is verified by the signing scheme with the hash and public key.   

Messages must contain an account number, to look up the `signerPubKey` on-chain, and a `timestamp`, to order messages. Timestamps are client provided, unverified and should be considered best-effort, since they are vunlerable to [clock skew](https://en.wikipedia.org/wiki/Clock_skew) and [clock drift](https://en.wikipedia.org/wiki/Clock_drift). Users who want to ensure perfect ordering can use [hybrid clocks](https://martinfowler.com/articles/patterns-of-distributed-systems/hybrid-clock.html) to generate timestamps.

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
  }
};
```

Object serialization must be done according to [RFC-8785](https://datatracker.ietf.org/doc/html/rfc8785), hashing must be performed with [BLAKE2b](https://www.rfc-editor.org/rfc/rfc7693.txt) and signing must be performed with an Ed25519 signature scheme, or in rare cases an ECDSA secp256k1 scheme. 


## 2.3 Applications
An application is a program that people use to interact with the Farcaster network. It can be standalone client that can be downloaded and run locally, or hosted software that runs on a cloud server and can be accessed through a specific client. Users are free to choose the type of application that best suits their needs. 

A simple application might consist of a standalone desktop or mobile client that talks directly to a Farcaster Hub. It can publish new messages and view messages published by other accounts. Each client will need to be instantiated with a [valid signing key](#42-signer). 

A more sophisticated application might introduce a proxy backend server between the Hub and the client. Advanced features like search can be implemented on this server. It might also operate as a hosted service, where it custodies keys for the user and performs all the signing on the backend.

## 2.4 Hubs
A Hub is an always-on server that validates, stores, and replicates Signed Messages. Applications store data on Hubs to make them available to their followers at all times.

Hubs can fetch data from other Hubs, making it easier for mobile clients who tend to go offline a lot. For instance, Alice follows Bob and Charlie, who use separate Hubs. She configures her Hub to sync with their Hubs periodically. When her client comes online, it can fetch everyone's messages with a single call to her Hub.

Every Hub monitors the Account Contract so that it can validate signed messages, and detect malicious Hubs that send forged data. Therefore, we can trust data about any user from any Hub because self-authentication prevents forgery. If Bob has a copy of Charlie's messages, Alice's server can download them and save a round trip to Charlie's Hub. Hubs can use a gossip protocol and fetch data from the closest peer with a copy instead of going to the user's actual Hub.

Hubs form an **L2 network for storing social data**, though the network has different properties from blockchain-based L2s. Its consensus model has weaker consistency guarantees but stronger scalability guarantees because the network data is **shardable** down to the account level. Users can run their own Hub, share one with others or use a third-party service to host one on their behalf.

# 3. Identity System

The identity system ensures secure and decentralized ownership of user accounts. Users must be able to set up a new account without any third party approval. It should also be reasonably quick and cheap. For our purposes, we define this as being able to complete registration in under a minute and for less than $10. It is is also important that accounts are trustworthy and operated primarily by honest, active users. 

Farcaster's Identity System is composed of two smart contracts - an **Account contract**, which creates new accounts, and a **Namespace contract** which issues usernames that can be used with accounts.

## 3.1 Account Contract

An account is created by calling `register` on the contract using a user-controlled Ethereum address. A new account number is issued and linked to this address. Accounts can be transferred between addresses, though the contract ensures that an address owns no more than one account at a time. 

Account numbers start at 0 and are incremented by 1 every time a new account is registered, which is a gas efficient way to ensure unique account numbers. The account number is represented internally as a `uint256` counter and can be incremented up to ~ 10^77 which is practically infinite for our needs. 

## 3.2 Namespace Contract

A Farcaster Name like `@alice` can be minted from the Namespace contract and used on the protocol. 

When minting a name, the contract checks that the name is unique and contains at most 16 alphanumeric characters or dashes `^[a-zA-Z0-9-]{1,16}$`. The single dash (-) username is reserved for a system account and will not be claimable. The mint happens over a two-phase commit reveal to prevent frontrunning registrations.

Farcaster Names are ERC-721 tokens that are fully composable with the NFT ecosystem. While users can use ENS names with the protocol, Farcaster Names have some properties that make them more practical. Names are cheaper to mint and can be [recovered](#recovery) if the address holding them is lost. They are also less vulnerable to [homoglyph attacks](https://en.wikipedia.org/wiki/IDN_homograph_attack) and are easier to display because of the restricted length and character set. 

A yearly fee of 0.01 ETH is charged for owning a name, and this must be renewed yearly on Farcaster Day (August 1st). Users who do not renew their usernames by this day have a 30-day grace period after which their ownership over the name expires. During mint, the fee is pro-rated for the year ending August 1st. Fees are collected by the Farcaster Treasury and are used to support protocol development.

Names can be minted freely, but are subject to two policies enforced by governance: 

1. During the first year, a "prior ownership" claim can be made if the user already owns the name on at least two major social media platforms, and the current owner does not own them on any platform. 

2. During later years, community moderators can propose an "impersonation claim" to governance vote if someone is clearly impersonating or misleading users. 

Names that expire or subject to a claim are moved back into the Farcaster Treasury, which may choose to transfer ownership to someone with "prior ownership" claims or auction them to the highest bidder. For impersonation claims, any fees that were previously paid are forefit.

## 3.3 Recovery

Names and Addresses can be recovered if user loses the keys to the address holding them. Both contracts implement a time-delayed recovery system that allows a **recovery address** to request a transfer to a new address. If the custody address does not cancel the transfer within 3 days, the recovery address can complete the transfer. 

Users can set the recovery address to another address in their wallet, a multi-sig shared with friends, or a third-party recovery service. Users can also change the recovery address at any time. Ownership remains decentralized because the recovery address cannot make a transfer that the custody address does not approve. 

Transferring the asset to a new custody address must unset the recovery address. Otherwise, users may purchase a name on OpenSea only to have the previous owner claim it back stealthily with their recovery address.

# 4. Replication

Replication is the process by which Hubs accept new messages and determine a user's state. 

Users send [messages](#41-self-authenticated-message) to a Hub for every action they take. If a user likes a URL, unlikes it, and likes it again, that creates three messages. A Hub that receives all messages will determine the current state of the URL as *liked by the user*. The Hub discards the first two messages to save space since they are no longer needed. Merging messages at the Hub level avoids client disagreements on state and saves space.

Every message type will have different rules for the merge operation. For example, two likes on the same cast by the same user can be condensed into one, while two replies cannot. Hubs implement a Set for each message type, which is a [conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) that encodes specific validation and merge rules.

Sets ensure [strong eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency#Strong_eventual_consistency) so that two Hubs that receive the same messages over any period will always reach the same state. This property makes Hubs highly available since they can go offline at any time and always get back into sync. Formally, Sets are anonymous Δ-state CRDTs[^delta-state], and each message is a join-irreducible update on the set. 

<!-- Diagram of all user data types -->

## 4.1 Signers

A Signer is a message that authorizes a new keypair to sign messages on behalf of an account. 

```ts
type SignerMessage = {
  active: boolean;
  signerPublicKey: string;
  schema: 'farcaster.xyz/schemas/v1/signer';
};
```

When a Farcaster account is created its `custody address` is the only address that can sign messages on its behalf. It is known as a Root Signer, and its authority can be verified on-chain. A Root Signer can authorize Delegate Signers by creating a Signer message, and Delegate Signers can authorize more Delegates. If an account moves to a new custody address, this also becomes a valid Root Signer.  Valid signers can be represented as a series of trees, where each tree's root is a historical custody address. 

Delegate signers must be Ed25519 keypairs and all signatures on the Farcaster network must be signed with this scheme. The only exceptions are signer messages created by root signers, which are ECDSA secp256k1 key pairs out of necessity. If a signer is compromised, it can be revoked by itself or any of its ancestors in the tree. All messages created by the signer and its children will be discarded, because there is no way to tell the user's messages from the attackers. 

<!-- Diagram of Signer Tree -->

The Signer Set is a is a modified two-phase set[^two-phase-set] with a combination of remove-wins and last-write-wins semantics.  New messages are added into the set if signed by a valid delegate or root signer. A remove message is accepted if signed by itself or an ancestor. Once removed it can never be re-added, and it's child signers and messages signed by them are removed. 

A conflict can occur if two parents add a message with the same `publicKey` in the message. This cannot be reconciled with the tree structure since it creates ambiguity about ancestry. If seen two such messages are seen, the set keeps the one with the highest timestamp and lexicographical hash, in that order. 

## 4.2 Casts

A Cast is a public message created by a user that is displayed on their profile.  It can contain text as well as links to media, on-chain activity or other casts. Users are allowed to delete casts at any time. 

Casts are managed using a two phase set, which has two collections: an **add set** and a **remove set**. New casts are placed in the add set and moved to the remove set when deleted by the user. Once a message is deleted, it can never be moved back into the add set. The Cast Set can be said to have "remove wins" semantics for the merge operation. 

Casts come in several different flavors and the protocol can be extended to support many more types in the future. Each type is stored in it's own two-phase set which may have additional rules for merging new messages into the set. 

### 4.2.1 Short Text Casts
A short public post created by a user that can appear directly on their profile, as a reply to another cast, url or on-chain item. Short casts can have upto 280 unicode characters and two embeds. The `parentUri` property can reference any URI, except for itself or its children. 

```ts
type CastShortTextBody = {
  embed: Embed;
  text: string;
  schema: 'farcaster.xyz/schemas/v1/cast-short-text';
  parentUri?: URI;
};
```

### 4.2.2 Recasts
A recast expresses an intent to share another cast. The set ensures that a recast message does not reference itself, and that only one recast exists for each value of `account` and `targetCastUri`. If multiple values are discovered, it keeps the one with the highest timestamp and highest lexicographical order, in that order. 

```ts
type CastRecastMessageBody = {
  targetCastUri: URI;
  schema: 'farcaster.xyz/schemas/v1/cast-recast';
};
```

### 4.2.3 Deletes
A delete instructs the set to remove a previously created cast. It is a type of soft delete where the content of the message is removed but its hash is retained forever. A user who has a copy of the deleted message can prove that the message was posted at some point by the author. 

The delete message must contain the hash of the cast being removed and omit all the other properties. The set can then remove the cast and its contents from the network, which is desirable from a user perspective. The set ensures that the delete message does not reference itself, and that it is only applied as a remove operation if it has a timestamp higher than that of the message it references. 

```ts
type CastDeleteBody = {
  targetCastHash: string;
  schema: 'farcaster.xyz/schemas/v1/cast-delete';
};
```

### 4.2.4 Embeds
A data structure used to embed URIs within a cast. Clients should hydrate these URIs and show an inline preview when rendering the cast in the UI. 

```ts
type Embed = {
  items: URI[];
};
```

## 4.3 Actions

An action is a public operation performed by the user on a target, which can be another user, cast or on-chain activity. Two types of actions are supported today: **likes** and **follows**. The protocol can be extended to support new actions easily. Users can undo and redo actions at any time. Conceptually, each action is an edge of the social graph of the Farcaster network.

Actions are managed with a Set that loosely resembles a [LWW-Element-Set CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#LWW-Element-Set_(Last-Write-Wins-Element-Set)). Only one action in the set can have the same `type` and `targetUri`. If a second action is seen, the one with the highest timestamp is retained. If the timestamps are equal, the one with the lowest lexicographical hash order is retained.

```ts
type ActionMessageBody = {
  active: boolean;
  type: 'like' | 'follow'
  targetUri: FarcasterURI;
  schema: 'farcaster.xyz/schemas/v1/action';
};
```

## 4.4 Verifications

This section is still in progress, and will cover a CRDT for verified data types, including proof of ownership of an address or a specific NFT.s 

## 4.5 Metadata

This section is still in progress, and will cover a CRDT for allowing arbitrary metdata to be added to a user's account like a display name or profile picture. 

## 4.6 Root Revocations

A Root Revocation is a special message that is used to remove previous custody addresses from the list of valid signers. This is useful if you believe that a previous address may have become compromised or if you are changing ownership of an account. 

A revocation message must include the blockchash of a specific Ethereum block. It must be signed by a custody address that owned it at the end of that block or afterwards. When received, the custody address at the end of the block specified is considered the first valid root signer. All previous custody addresses and delegate signers issued by them are invalidated.

```ts
type RootRevocationBody = {
  blockHash: string;
  schema: 'farcaster.xyz/schemas/v1/root-revocation';
}
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

A malicious attacker can use a compromised delegate signer to impersonate the user by signing messages. As long as the user has control over a parent signer or the root signer, they can invalidate the signare and issue a new one. Unfortuantely, this means that all messages signed by that signer will be lost permanently since we cannot tell which ones were signed by the attacker. Clients can mitigate this by constantly rolling keys after every few thousand messages, which limits the scope of how many messages will be lost when a signer is reset.

## 7.2 Eclipse Attacks
A malicious user could spin up several Hubs which pretend that a target user has published zero messages. Peers might assume this to be true, effectively blocking the target from the network. Hubs can maintain an internal score for each peer based on data availability. Periodically, they lookup the location of the user's source Hub which is published in the account contract and sync the latest messages. If their peers do not have an up-to-date copy of this, their scores are lowered until eventally the peer is dropped and a new one is selected.  

## 7.3 Flooding Attacks 
A flooding attack is when a malicious user keeps acquiring new accounts and broadcasting thousands of messages. Hubs may attempt to sync this data bloating their on-disk storage and causing network congestion which leads to stale data for legitimate user accounts. An account scoring system can help alleviate this by prioritizing trustworthy accounts and temporarily or permanently banning misbehaving accounts. Each Hub tracks its own score for an account which starts at zero. The score increases if the account has a valid username and a history of behaving well. It decreases when malicious behavior like flooding is obsered.

## 7.4 DDOS Attacks
A DDOS attack is when a malicious Hub spams a target Hub with queries that are expensive to run causing it to stop responding to legitiamate requests and syncing with other Hubs. A simple mitigation strategy is to implement IP-based rate limiting that rejects query requests from Hubs when they exceed a threshold. More sophisticated DDOS attacks might require aggressive mitigations like whitelisting a set of known peers or relying on infrastructure-level DDOS protection services offered by cloud vendors.  

## 7.5 Replay Attacks
A replay attack is when a malicious actor stores a copy of a user's message and is able to replay it to cause an action that the user did not intend. This attack is possible if a Signer is removed, since it may delete a message that causally removed a prior message from the state. Assume Alice added a "hello world" cast with Signer A and then deleted it with Signer B and then proceeded to delete Signer B. At this point, the cast is no longer present on the Hub but neither is the delete message. A malicious user with a copy of the "hello world" cast could replay it causing it to be accepted as valid. Fortunately, the attack is restricted to replaying previously valid messages and  Alice can issue a remove message from a currently valid signer to nullify it. 

# 8. URIs

This section is still under development and will cover a schema for URIs supported by Farcaster Message types.

# 9. Governance

Farcaster is a decentralized protocol that is not controlled by a single individual, and governance is the process of making protocol changes in a decentralized way. The process is kept lightweight during beta to encourage community contributions and rapid development cycles.

Anyone can propose a change by opening up a [new discussion topic](https://github.com/farcasterxyz/protocol/discussions) in the protocol repository. The Farcaster team will provide feedback and and may ask for more details. Once all feedback has been provided the Farcaster team will decide whether to include the change in the roadmap or whether to reject it. Once approved, an issue is created and the specification changes are merged into this repository. 

## Hub Changes

Changes that involve off-chain systems must be implemented and deployed in the Hubs. The Farcaster team will work closely with Hub developers and operators to pick a release date that will ensure a smooth transition. The Farcaster team is also reponsible for ensuring that there is strong alignment around implementing the changes. 

Developers and operators can veto a change if they disagree with it, but at some cost to themselves and the network. An operator may choose not to upgrade their Hub version and a developer can choose not to release the change. This will cause fragmentation and users on such Hubs may not be visible to the rest of the network. It is desirable for developers and operators to have this power to ensure decentralization of the network, but ideally they would never need to exercise it.

## Contract Changes

Changes that involve on-chain systems must be implemented by deploying a new contract or upgrading an existing one. The Farcaster team will implement these changes and ensure that they are thoroughly audited. Contracts will be controlled by a multi-sig whose ownership is split between members of the Farcaster team during beta. Over time, control over making contract changes will be decentralized to other parties who have a vested interested in ensuring the success of the network. 

[^delta-state]: van der Linde, A., Leitão, J., & Preguiça, N. (2016). Δ-CRDTs: Making δ-CRDTs delta-based. Proceedings of the 2nd Workshop on the Principles and Practice of Consistency for Distributed Data. https://doi.org/10.1145/2911151.2911163

[^two-phase-set]: Shapiro, Marc; Preguiça, Nuno; Baquero, Carlos; Zawirski, Marek (2011). "A Comprehensive Study of Convergent and Commutative Replicated Data Types". Rr-7506.