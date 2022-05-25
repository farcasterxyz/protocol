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

## Identifiers

## Usernames

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
