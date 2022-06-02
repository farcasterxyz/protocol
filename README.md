# Farcaster Protocol


## Contents
1. [Introduction](#1-introduction)
2. [Concepts](#2-concepts)
    1. [Accounts](#21-accounts)
    2. [Signed Messages](#22-signed-messages)
    3. [Applications](#23-applications)
    4. [Hubs](#24-hubs)
3. [Identity System](#3-identity-system)
    1. [Account Registry](#31-account-registry)
    1. [Namespace Contract](#32-namespace-registry)
    3. [Recovery](#33-recovery)
4. [Replication](#4replication)
    1. [Casts](#42-casts)
    2. [Actions](#43-actions)
    3. [Verifications](#44-verifications)
    4. [Metadata](#45-metadata)
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

Farcaster is a [sufficiently decentralized](https://www.varunsrinivasan.com/2022/01/11/sufficient-decentralization-for-social-networks) protocol that empowers developers to build novel social networks. We define a sufficiently decentralized network as one where two users who want to communicate are always able to, even if the network wants to prevent it. Users on such networks must have complete control over their identity (usernames), data (messages), and social graph (relationships to others). If a third party controls any of these, they can prevent two users from communicating. Developers must also be free to build applications and have unrestricted access to the network, and users must be free to switch between them. If there was only one app to connect to the network, it could prevent two users from communicating.

# 2. Concepts

Farcaster achieves sufficient decentralization through a hybrid architecture with on-chain and off-chain components.

The protocol stores user identities on  Ethereum to leverage its robust security, composability, and strong consistency guarantees. An Ethereum address controls the on-chain identity and can be used to sign off-chain messages on behalf of the identity. 

Ethereum and other L1 blockchains cannot practically store the large volume of data that users generate. Instead, users store their data off-chain on a server under their control known as a Farcaster Hub. The user's identity address must cryptographically sign all data before sending it to the Hub.

<!-- Diagram covering the major architectural concepts  -->

## 2.1 Accounts

A Farcaster account is similar to an account on pseudonymous social networks like Twitter or Reddit. An individual can operate several accounts simultaneously, like a real-name account, a pseudonymous account, and a company account. 

An address can mint a new account from the AccountRegistry, which issues it an account number. This address is known as the custody address, and it can sign messages on behalf of the account. Accounts can be enriched by adding a profile picture, display name, biography, and verified usernames like `alice.eth`, which are all done off-chain with signed messages.

## 2.2 Signed Messages

Signed Messages are objects that are **tamper-proof** and **self-authenticating**.

A Signed Message has a **message** property that contains the payload. The payload is then serialized, hashed, and signed with the custody address. The **envelope** property contains this hash, signature, and the public key of the custody address. It can be used to verify the message's authenticity with the following steps: 

1. Look up the custody address of the account number and verify that its public key matches the envelope's `signerPubKey`.
2. Serialize and hash the message and verify that it matches the envelope's `hash`.
3. Verifying the signature scheme with the envelope's `signature`, `hash`, and `signerPubKey`.

The message must be serialized with [RFC-8785](https://datatracker.ietf.org/doc/html/rfc8785), hashed with [BLAKE2b](https://www.rfc-editor.org/rfc/rfc7693.txt) and signed with an Ed25519 signature scheme. Each message must also contain an account number to look up the custody address on-chain and a timestamp for ordering. Timestamps are client-provided, unverified, and should be considered best-effort since they are vulnerable to [clock skew](https://en.wikipedia.org/wiki/Clock_skew) and [clock drift](https://en.wikipedia.org/wiki/Clock_drift). Users who want to ensure perfect ordering can use [hybrid clocks](https://martinfowler.com/articles/patterns-of-distributed-systems/hybrid-clock.html) to generate timestamps.

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

## 2.3 Applications

An *application* is a program that people use to interact with the Farcaster network. Users can choose the type of application that best suits their needs and switch between them at any time. 

A simple application might consist of a standalone desktop or mobile client that talks directly to a Farcaster Hub. It can publish new messages and view messages published by other accounts. Such applications are **self-hosted** and must be instantiated with the custody address or [valid signing key](#42-signer).

A more sophisticated application might add a proxy backend server that indexes data from Hubs. Indexing allows servers to implement features like search, algorithmic feeds, and spam detection that are difficult or expensive to perform on the Hub. Such applications can be **self-hosted** by storing keys on the client; **delegated** by asking users for a [delegate signing key](#42-signer); or **hosted** by managing all keys including the custody address.

## 2.4 Hubs
A Hub is an always-on server that validates, stores, and replicates Signed Messages. 

Users must upload messages they create to a primary Hub and publish its URL on-chain using the AccountRegistry. Their followers can use this Hub to find and download their messages. Users can run a Hub themselves or use a third-party Hub service. They are incentivized to ensure that it works correctly, or their followers will not receive their messages.

Users can also configure their primary Hub to replicate data from other Hubs. If Alice follows Bob and Charlie, who use separate Hubs, she can configure her Hub to download messages from theirs. When her client comes online, it can make a single request to her Hub and fetch Bob and Charlie's messages. 

Hubs maintain a connection to the Account Registry to validate every Signed Message they receive. A malicious Hub that served a forged message would be detected because the message authentication would fail. This property of Signed Messages lets us safely receive messages signed by *any* user from *any* Hub. If Bob has a copy of Charlie's messages, Alice's server can download them and save a round trip to Charlie's Hub. Hubs can fetch data from nearby peers using a gossip-based pubsub protocol [^gossip-sub] instead of making a round trip to each user's primary Hub.

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

1. A *prior ownership* claim can be made by a user who owns a name on at least two major social media platforms, if the current owner does not own the name on any major social media platforms. 

2. An *impersonation claim* can be made if someone is clearly impersonating or misleading users, and it will be investigated by a group of moderators elected by the community though a governance vote.

Names that expire or are subject to a claim are moved back into the Farcaster Treasury. The treasury can choose to transfer ownership to someone with ownership claims or auction them to the highest bidder. For impersonation claims, any previously paid fees are forfeit.

## 3.3 Recovery

Names and Addresses are recoverable if the user loses the keys to the address holding them. Both contracts implement a time-delayed recovery system that allows a **recovery address** to request a transfer to a new address. If the custody address does not cancel the transfer within three days, the recovery address can complete the transfer. 

Users can set the recovery address to another address in their wallet, a multi-sig shared with friends, or a third-party recovery service. Users can also change the recovery address at any time. Ownership remains decentralized because the recovery address cannot make a transfer that the custody address does not approve. 

Transferring the asset to a new custody address must unset the recovery address. Otherwise, users may purchase a name on OpenSea only to have the previous owner claim it back stealthily with their recovery address.

# 4. Replication

*Replication* is the process by which Hubs accept new messages and determine a user's state. 

Users send [messages](#41-self-authenticated-message) to a Hub for every action they take. If a user likes a URL, unlikes it, and likes it again, that creates three messages. A Hub that receives all messages will determine the current state of the URL as *liked by the user*. The Hub discards the first two messages to save space since they are no longer needed. Merging messages at the Hub level avoids client disagreements on state and saves space.

Every message type will have different rules for the merge operation. For example, two likes on the same cast by a user can be condensed into one, while two replies cannot. Hubs implement a Set for each message type, which is a [conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) that encodes specific validation and merge rules.

Sets ensure [strong eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency#Strong_eventual_consistency) so that two Hubs that receive the same messages over any period will always reach the same state. This property makes Hubs highly available since they can go offline at any time and always get back into sync. Formally, Sets are anonymous Δ-state CRDTs[^delta-state], and each message is a join-irreducible update on the set. 

<!-- Diagram of all user data types -->

## 4.1 Casts

A *Cast* is a public message created by a user that is displayed on their profile.  It can contain text as well as links to media, on-chain activity or other casts. Users are allowed to delete casts at any time. 

Casts are managed using a two phase set, which has two collections: an **add set** and a **remove set**. New casts are placed in the add set and moved to the remove set when deleted by the user. Once a message is deleted, it can never be moved back into the add set. The Cast Set can be said to have "remove wins" semantics for the merge operation. 

Casts come in several different flavors and the protocol can be extended to support many more types in the future. Each type is stored in it's own two-phase set which may have additional rules for merging new messages into the set. 

### 4.1.1 Short Text Casts
A short public post created by a user that can appear directly on their profile, as a reply to another cast, url or on-chain item. Short casts can have upto 280 unicode characters and two embeds. The `parentUri` property can reference any URI, except for itself or its children. 

```ts
type CastShortTextBody = {
  embed: Embed;
  text: string;
  schema: 'farcaster.xyz/schemas/v1/cast-short-text';
  parentUri?: URI;
};
```

### 4.1.2 Recasts
A recast expresses an intent to share another cast. The set ensures that a recast message does not reference itself, and that only one recast exists for each value of `account` and `targetCastUri`. If multiple values are discovered, it keeps the one with the highest timestamp and highest lexicographical order, in that order. 

```ts
type CastRecastMessageBody = {
  targetCastUri: URI;
  schema: 'farcaster.xyz/schemas/v1/cast-recast';
};
```

### 4.1.3 Deletes
A delete instructs the set to remove a previously created cast. It is a type of soft delete where the content of the message is removed but its hash is retained forever. A user who has a copy of the deleted message can prove that the message was posted at some point by the author. 

The delete message must contain the hash of the cast being removed and omit all the other properties. The set can then remove the cast and its contents from the network, which is desirable from a user perspective. The set ensures that the delete message does not reference itself, and that it is only applied as a remove operation if it has a timestamp higher than that of the message it references. 

```ts
type CastDeleteBody = {
  targetCastHash: string;
  schema: 'farcaster.xyz/schemas/v1/cast-delete';
};
```

### 4.1.4 Embeds
A data structure used to embed URIs within a cast. Clients should hydrate these URIs and show an inline preview when rendering the cast in the UI. 

```ts
type Embed = {
  items: URI[];
};
```

## 4.2 Actions

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

## 4.3 Verifications

This section is still in progress, and will cover a CRDT for verified data types, including proof of ownership of an address or a specific NFT.s 

## 4.4 Metadata

This section is still in progress, and will cover a CRDT for allowing arbitrary metdata to be added to a user's account like a display name or profile picture. 

## 4.5 Signer Authorizations

A *Signer Authorization* is a message that authorizes a new key pair to generate signatures for a Farcaster account. 

When an account is minted, only the custody address can sign messages on its behalf. Users might not want to load this keypair into every device since it increases the risk of account compromise. The custody address, also known as the *Root Signer*, can authorize other keypairs known as *Delegate Signers*. Unlike Root Signers, a Delegate Signer is only allowed to publish off-chain messages and cannot perform any on-chain actions. 


Root Signers generate ECDSA signatures on the secp256k1 curve and can only publish Signer Authorization messages. All other types of messages must be signed by Delegate Signers, which creates EdDSA signatures on Curve25519[^ed25519]. Delegate Signers can be used to authorize new devices or even third-party services to sign messages for an account. If a Delegate Signer is compromised, it can be revoked by itself, an ancestor in its chain of trust, or any Root Signer. When a Signer is revoked, Hubs discard all of its signed messages because there is no way to tell the user's messages from the attackers.

Users might also transfer an account to a new custody address due to key recovery or changing wallets. It is usually desirable to preserve history and therefore both custody addresses become valid Root Signers. The set of valid signers for an account form a series of distinct trees. Each tree's root is a historical custody address, and the leaves are delegate signers.

<!-- Diagram of Signer Tree -->


The Signer Set is a modified two-phase set[^two-phase-set] with remove-wins and last-write-wins semantics. New messages are added to the set if signed by a valid delegate or root signer. A remove message is accepted if signed by itself or by an ancestor. A Signer can never be re-added once removed, and all of its descendant children and messages are discarded.

A Set conflict can occur if two valid Signers separately authorize the same Delegate Signer, which breaks the tree data structure. If this occurs, the Set retains the message with the highest timestamp and lexicographical hash, in that order.

```ts
type SignerMessage = {
  active: boolean;
  signerPublicKey: string;
  schema: 'farcaster.xyz/schemas/v1/signer';
};
```

## 4.6 Root Signer Revocations

A Root Signer Revocation is a special message that is used to remove previous custody addresses from the list of valid signers. This is useful if you believe that a previous address may have become compromised or if you are changing ownership of an account. 

A revocation must include the blockchash of a specific Ethereum block. It must be signed by a custody address that owned it at the end of that block or afterwards. When received, the custody address at the end of the block specified is considered the first valid root signer. All previous custody addresses and delegate signers issued by them are invalidated.

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
A malicious user could spin up several Hubs which pretend that a target user has published zero messages. Peers might assume this to be true, effectively blocking the target from the network. Hubs can maintain an internal score for each peer based on data availability. Periodically, they lookup the location of the user's source Hub which is published in the AccountRegistry and sync the latest messages. If their peers do not have an up-to-date copy of this, their scores are lowered until eventally the peer is dropped and a new one is selected.  

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

[^gossip-sub]: Dimitris Vyzovitis, Yusef Napora, Dirk McCormick, David Dias, Yiannis Psaras: “GossipSub: Attack-Resilient Message Propagation in the Filecoin and ETH2.0 Networks”, 2020; [http://arxiv.org/abs/2007.02754 arXiv:2007.02754].

[^delta-state]: van der Linde, A., Leitão, J., & Preguiça, N. (2016). Δ-CRDTs: Making δ-CRDTs delta-based. Proceedings of the 2nd Workshop on the Principles and Practice of Consistency for Distributed Data. https://doi.org/10.1145/2911151.2911163

[^two-phase-set]: Shapiro, Marc; Preguiça, Nuno; Baquero, Carlos; Zawirski, Marek (2011). "A Comprehensive Study of Convergent and Commutative Replicated Data Types". Rr-7506.

[^ed25519]: Bernstein, D.J., Duif, N., Lange, T. et al. High-speed high-security signatures. J Cryptogr Eng 2, 77–89 (2012). https://doi.org/10.1007/s13389-012-0027-1

