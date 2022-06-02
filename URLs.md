# Farcaster URLs

Farcaster Messages may reference other messages, online content, or on-chain activity. An extension of the URL standard is employed by Farcaster to identify all resources and activities that can be referenced within protocol messages. 

## Requirements

- *Extensible*: The standard should anticipate the addition of new resource types.
- *Versionable*: The standard should accommodate new and evolving requirements for existing resource types while preserving support for older identifiers
- *Forward Compatible*: The addition of new resource and activity types to the standard should not require a coordinated upgrade of all Farcaster nodes and clients. The standard should allow for clear differentiation between invalidly constructed identifiers and identifiers that employ the use of not-yet-supported resource types.
- *Canonical Representation*: Canonicalization is enforced for both Farcaster and chain-based resources. Web-hosted resources cannot be expected to conform to a single canonical URL and are excluded from this constraint.

## Specification

The standard is an extension of the common [URL standard](https://datatracker.ietf.org/doc/html/rfc1738) with support for the HTTP, HTTPS, and [IPFS](https://docs.ipfs.io/how-to/address-ipfs-on-web/#native-urls) protocols as well as the newly introduced `chain` and `farcaster` schemes.

```
<protocol>://<resource>,  <= 2048 characters

protocol      =  "https" | "http" | "ipfs" | "chain" | "farcaster"
resource      = (defined by each protocol specification)
```

The Farcaster URL structure adds the following additional protocols to the URL prefix:

- `chain://`
- `farcaster://`

### Chain Resources

Resources and activity across different networks will be identified by a URL with the `chain:` protocol, followed by the network's [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) chain ID.

Following the chain ID may be a [CAIP-10](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md) account specifier, a [CAIP-19](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-19.md) asset specifier, a transaction identifier, or other unspecified identifiers for on-chain resources and activity.

```
chain://<network>
chain://<network>:<address>
chain://<network>/<asset_namespace>:<asset_reference>/...
chain://<network>/tx:<tx_identifier>
chain://<network>/<future_placeholder>

network             = CAIP-2 chain id
address             = an address string in the common format of the given network
asset_namespace     = an identifier for a class of related assets, such as erc20 and slip44
asset_reference     = identifies an asset within its asset_namespace
tx_identifier       = a transaction identifier in the common format of the given network
future_placeholder  = future message types
...                 = addition url parameters for the standard (optional)
```

Examples:

```
chain://eip155:1:0xaba7161a7fb69c88e16ed9f455ce62b791ee4d03
chain://eip155:1/erc721:0xaba7161a7fb69c88e16ed9f455ce62b791ee4d03/7894
chain://bip122:000000000019d6689c085ae165831e93/slip44:0
```

While the protocol allows for the introduction of new on-chain resources, the minimal set implemented by a node or a client must include:

- Ethereum addresses
- Ethereum transactions
- Ethereum ERC20 contracts
- Individual Ethereum ERC-721 tokens
- Individual Ethereum ERC-1155 tokens

### Farcaster Resources

Farcaster URLs must be capable of identifying any message signed by a user. A sample URL might look like `farcaster://id:8789213729/cast:0xf00b4r` and is of the form:

```
farcaster://<user>
farcaster://<user>/<message_type>:<message_id>

user                = id:<ID>
ID                  = uint256
message_type        = "cast" | "reaction" | "verified_address" | "follow" | <future_placeholder>
message_id          = unique identifier for the message
future_placeholder  = future message types
```

Example:

```
farcaster://id:8789213729/cast:0xf00b4r
```

## Versioning

If an existing resource or activity type already represented by Farcaster URLs must be updated for any reason, the same URL structure must continue to be supported alongside its upgraded version. A hyphenated suffix should describe the version upgrade. For example:

```
farcaster://id:8789213729/cast-v2:0xf00b4r/1652945621
```

might represent an update to the URL schema Farcaster Casts that adds a unix epoch timestamp to follow `<message_id>` in the original schema.

Any namespace identifier in the URL schema may be suffixed with a version update, as can the protocol identifier itself if necessary (e.g. `farcaster-v2://`).

## Extending

The Farcaster URL standard may be extended by the community via updates to this document and to the URL parser provided by the reference Farcaster node. The hierarchical nature of URLs allows for the addition of new identifier types at any point in the schema.

## Forward Compatibility

Farcaster node and client software should take care not to reject as invalid any well-formed URLs that make use of unrecognized namespaces and identifier types. As an example, the URL `widget://1` is a well-formed URL that makes use of an unrecognized `widget` schema identifier and should be treated as valid but un-recognized. Whereas the URL `farcaster://id:8789213729/cast:1-1` is known to be invalid, as the type of resource being identified is understood as a Farcaster Cast but `1-1` is not a valid Cast identifier.
