# Farcaster Protocol

Farcaster is a sufficiently decentralized social networking protocol where users can publish messages to followers. Anyone can build a Farcaster application and users are free to switch between them. Unlike other social networks, Farcaster gives users control over their username and developers access to the social graph. 

## Contents
1. [Introduction](#introduction)
2. [Identity](#identity)
    1. [Identification Number](#identification-number)
    2. [Name](#name)
3. [Hub]()
    1. [Storage]()
    2. [Transport]()
    3. [Peer Discovery]()
    4. [API]()
    5. [Ethereum Sync]()
    6. [Upgradeability]()
4. [Data]()
    1. [Signing]()
    2. [CRDTs]()
    3. [Types]()
        1. [Signer]()
        2. [Root]()
        3. [Cast]()
        4. [Link]()
        5. [Meta]()
    4. [URIs](#uris) 
    5. [Upgradeability]()
5. [Applications]()

# Introduction

Social networks have become an essential part of our lives over the last decade. Many of these networks began their journey as open platforms, courting developers to build on their APIs. These developers created new clients, discovered new UI paradigms, and even launched multi-billion dollar businesses that brought in many users.

Today, most networks have given up on the idea of being an open platform. APIs have been pared back, and no social network will allow anything resembling an alternate client to connect to the network. Builders with new ideas for monetization, moderation, signaling, or recommendation systems have no path to realizing them.

We believe that a [sufficiently decentralized](https://www.varunsrinivasan.com/2022/01/11/sufficient-decentralization-for-social-networks) protocol for social networking will empower developers to build novel social networks. A protocol can be considered sufficiently decentralized if and only if **two users who want to communicate are always able to, even if the network wants to prevent it**. If true, this implies that users have complete control over their data and can give developers access to it. Developers, in turn, are free to build applications and monetize them in any way they like. Most importantly, no user is ever locked into a particular application as long as they control their identity. 

Farcaster achieves sufficient decentralization by combining on-chain identities and off-chain signed data. New users must register a new identity on the Ethereum blockchain, owned by their address. They must then set up a server that stores their data off-chain known as a Farcaster Hub and publish its location on-chain. Finally, users can create new messages, sign them and store them on their Hub. Two users can always find each other by looking up the location of the other's Hub on the blockchain and downloading their messages. 

Hubs are a dynamic, peer-to-peer network of servers designed to store and replicate social data efficiently. Each Hub can choose the set of users it subscribes to and can change this at any time. They operate similarly to Bittorrent, and can download messages directly from the source or from a peer that has a copy. Since the user cryptographically signs each message, Hubs cannot tamper with messages as they move through the network.

# Identity

An identity system is an important consideration in social network design as it affects security, user experience, and social dynamics. Pseudonymous-name networks like Reddit encourage different user behaviors from real-name networks like Facebook. Our goal with Farcaster is to design an identity system that is: 

1. **Secure, decentralized, and pseudonymous**:  Users must be able to claim and use an identity on the network without third-party approval.

2. **User-friendly**: Acquiring an identity should be quick, cheap, and easy. In practical terms, users should be able to get started in under a minute for less than $10, without any special knowledge about blockchains or wallets.

3. **Trustworthy**: Squatting and impersonation of scarce username resources should be addressed to ensure that they are owned by honest, active users. 

On Farcaster, users establish identity by claiming a **Farcaster Identification Number (FIN)**. A FIN serves as a pointer to an account, similar to a primary key in a database. Users may have many social accounts and can get a FIN for each one. Users can optionally connect a FIN with an NFT username they own, like an ENS or [Farcaster name](#names). While identifiers can be acquired freely and in a decentralized manner, user namespaces are free to implement their own rules and regulations.

Applications should always reference a user account by its FIN and never by username or display name. The FIN is the only identifier that is guaranteed to be decentralized and always point to the same account. Applications can replace the FIN with the currently associated display or verified username at render time. User accounts become easier to recognize and harder to impersonate when displayed this way, and users still retain the ability to change their username in the future. This design meets our goals and, as a side effect, addresses the [Zooko's Triangle](https://en.wikipedia.org/wiki/Zooko%27s_triangle) naming trilemma. 

## Identification Number

A Farcaster Identification Number (FIN) is a deterministically generated number like `324234`. FINs are issued by the Farcaster Identity contract on the Ethereum blockchain, which ensures that an address can only hold one FIN at a time. FINs can be transferred between addresses, as long as this invariant is held. FINs begin at 0 and increment by 1 with every new identity issued, which is the most gas efficient way to issue unique identifiers on the EVM. They have a max size equal to the maximum value of a `uint256`, which is practically infinite for our needs. 

Once a FIN is acquired, the address that owns it can publish signed messages to Farcaster Hubs which will verify this signature. Such messages can be used to publish new posts, set a display name or even connect a verified ERC-721 username to the FIN. 

## Name

A Farcaster Name is a human-friendly name like `alice` that can be acquired from the Namespace contract. Names must be unique and can have up to to 16 alphanumeric characters or dashes `^[a-zA-Z0-9-]{1,16}$`. The single dash (-) username will be reserved for a special system account and will not be claimable. 

Names are issued as ERC-721 tokens that are fully composable with the NFT ecosystem. They are similar to prior systems like ENS, but have a few significant changes that make them more practical for use in a social network. For instance, names are restricted to a maximum of 16 ASCII characters with makes them easy to include in short messages and prevents [homoglyph attacks](https://en.wikipedia.org/wiki/IDN_homograph_attack). They are also cheaper to mint and have a [recovery system](#recovery) that protects users against key loss. 

Names have a registration fee of 0.01 ETH per year which is collected into the Farcaster Treasury. The fee can be paid on mint and must be renewed yearly transaction with a transaction on August 1st, also known as Farcaster Day. During mint, the fee is pro-rated for the year ending August 1st. Users who do not renew their usernames by this day have a 30-day grace period after which their ownership over the name expires. 

Names can be minted freely, but are subject to two policies enforced by governance: 

1. During the first year, a "prior ownership" claim can be made if the user already owns the name on at least two major social media platforms, and the current owner does not own them on any platform. 

2. During later years, community moderators can propose an "impersonation claim" to governance vote if someone is clearly impersonating or misleading users. For instance if someone acquires `@elonmusk` and pretends to actually be Elon. 

Names that expire or subject to a claim are moved back into the Farcaster Treasury, which may choose to transfer ownership to someone with "prior ownership" claims or auction them to the highest bidder. For impersonation claims, any fees that were previously paid are forefit.


## Recovery

Users can recover both FIN's and Farcaster Names if the keys to the custody address are lost. A previously appointed **recovery address** can issue a time-delayed transfer request. After a waiting period of 3 days, the recovery address can complete the transaction if the custody address did not cancel it. To use the system, users set up a recovery address with the smart contract, after which: 
 
1. The recovery address can request a transfer at any time. 
2. The custody address can cancel the transfer within three days.
3. After three days have passed, the recovery address can complete the transfer.  
4. The custody address can change the recovery address at any time.

Users can set the recovery address to a second address they control, a multi-sig shared with friends, or even a third-party recovery service. Since the recovery address can never move the asset if the custody address is active, it preserves the decentralized ownership property. 

Transferring the asset to a new custody address must unset the recovery address. Otherwise, users may purchase a token on OpenSea only to have the previous owner claim it back stealthily with their recovery address. Any ERC token contract can implement this time-delayed recovery mechanism. 


# Hub

## Storage

## Transport

## Peer Disovery

## API's

## Ethereum Sync

## Upgradeability

# Data 

## Signing

## CRDT's

## Types

## URIs

## Upgradeability

# Applications
