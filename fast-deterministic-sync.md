# Fast prefix sync

> A sync protocol that relies on the prefix nature of radix tries and consequently the Modified Merkle Patricia Trie. The main idea is the ability to offset into the trie using a prefix to select all data along a path starting with that prefix. It also has the advantage of working with both, full sub-tries (intermediary and leaf nodes) as well as just leaf nodes (accounts and storage entries).

## Overview

Prefix based chunking works something like this, pick some prefix length and use the radix of the trie to generate a range of chunk prefixes. For example, if we use 4 as the the prefix length we get 65536 (16^4) chunks or a range of 0x0000-0xffff. Knowing the prefixes ahead of time has the obvious advantage that we don't need to wait for a reply to request the next chunk, it can be done in parallel to multiple peers.

## Full sub-tries vs leaf nodes

A full sub-trie is a chunk with all the intermediary and leaf nodes along a prefix path meaning that the sub-trie is fully traversable along that prefix, and the validity of the chunk can be quickly verified. This has the advantage of discarding invalid chunks early on but increases on the wire overhead. 

Contrasted to delivering just the leafs, which doesn't have any additional on the wire overhead, but an attacker can feed invalid data before the client realizes and disconnects. 

I don't want to make any assumptions about this right now, but it seems like delivering just the leafs has the huge disadvantage of having to wait before the trie is sufficiently (fully?) reconstructed before realizing that there was something wrong. Given it's size and the time it could take to get there, it seems too big of an attack vector to ignore. (I'd love to be proven wrong)

### Compact proofs

Can the intermediary nodes, which are essentially proofs be compressed? Maybe some some sort of signature aggregation? I'm leaving that out for now as it seems like it is a bit of a premature optimization, but is worth exploring as the gains could be pretty big.

## How it works

The generalized flow is something like this:

(This assumes full sub-tries)

### "Client" flow

- pick a desired prefix size and generate the range of chunk roots to be requested
  - pick some prefixes and request them from the "server" for a given state or storage root
  - server responds with the chunks
    - all chunks delivered, repeat the flow for the next set of prefixes
    - the chunk did not fit in one response, but the server delivered what it could and the new prefix where the chunk stops
      - repeat the flow using the stop prefix above
  - error, no chunks found, move on to the next peer

### "Server" flow

- a chunk request arrived
  - extract the requested chunks from cache/db
    - chunks found
      - send the chunks back
    - one or several chunks too large
      - deliver what we can plus the path at which the chunk stops, so the client can pick up from there
    - error, no chunks for prefixes

#### A few observations

- The protocol is best effort
- It's adaptable, if the response/chunk is too large, chunk it some more
- Requests are decoupled
  - The client's next request doesn't depend on the previous response for the general case
  - In the case of partial responses, the client doesn't have to continue requesting from the same server
  - In "light" mode, the client might choose not to keep requesting remaining chunks even if it was a partial response
- The same protocol works for the state and storage tries
- All requests are tied to a state or storage root
- Requested prefixes could be sequential, range based, or even based on clients provided "accounts of interest" (which enables a type of "Light" mode in the client, more on this later)

### Several optimizations can be applied

- If the server doesn't have a chunk for the requested root/block, return error and the next best block for which the server does have the chunk
- A streaming mode can be added, where the server keeps replying with chunks for the prefix until EOF, so the client doesn't have to keep re-requesting at the stop path
- A flag can be added to the request, to indicate that the server can skip/include the intermediary nodes to reach the beginning of the chunk (the "stem") and only deliver nodes from where the chunk start (`key` - `prefix length`)
- The `stop path` can be used as a hint to generate another set of prefixes, e.g. if the stop path was `0x123456789`, instead of requesting just along the `0x12345678a` prefix, send a range [`0x12345678a`, `0x12345679f`], since it's best effort the server will send what it can (this might lead do some data overlap tho)
- A request with prefix `0x0`, can be used to pull the first few levels of the trie, and then use the stop path to determine the prefix length for subsequent request.

## Modified flow with `0x0` prefix

In full sub-trie mode, this allows retrieving the first few levels of the trie before hitting the maximum allowed chunk size. This has two advantages, 1) we can fill in the first few levels in one request right at the beginning, 2) it gives us an approximate prefix length to work with. 

For storage tries, in many cases this would be sufficient to retrieve the full trie. The large majority of contract storage doesn't exceed a few hundred KBs.

NOTE: in leaf mode, this would simply mean "give me all the leafs".

### "Client" flow

- make a request to "server" at some root, with prefix `0x0`
  - server responds with a chunk and a stop path
    - use the `key length - stop path length` to determine the prefix length, e.g. if the chunk is built up to level 3, use that as the prefix length
  - proceed as in the simplified flow above
  - error, move to the next peer

### "Server" flow

- a chunk request arrived, if the prefix is `0x0`
  - build a chunk from the root right up to where it becomes greater than `max chunk size`
    - return the chunk along with the stop path
  - error, no chunk available at root

## Retrieving only leafs (accounts and storage keys)

In general, this should "just work" for leaf nodes as apposed to full sub-tries as well. But it does assume that the clients can effectively retrieve accounts and storage keys based on prefixes. By this I mean that, if the node stores the accounts and optionally storage keys on some fast cache, that there is a way of quickly retrieving them from that cash based on prefixes without loosing the optimizations of the cache.

## The actual protocol

- `GetChunks` [`0x...`, `root`, [`prefixes`, ...], `isRange`, `maxSize`] - Request chunks of partial state
  - `root` - the state/storage root to use for the lookups
  - `prefixes` - the prefixes to request, can be a list, ranges or an individual prefix
  - `isRange` - flag to indicate if the prefixes is a range. If `true` prefixes are treated as range pairs and the total number of elements has to be even
  - `maxSize` - maximum size of the chunk in bytes the client is willing to accept, if the chunk is larger than this size, the server will stop building the chunk and return the path at which it stopped

- `ChunkData` [`0x...`, [[`prefix`, [[`hash`: `rlp data`, ...] || [`error`, [`code`, `data`]], `stopPath`]]] - Returns a set of previously requested chunks
  - `prefix` - can be either, an array of two elements; the first is a map of prefixes to node data, and the second an optional `stopPath`; otherwise its an `error` field with an error code and optional data
    - `map` & `stopPath`
      - each entry in the map corresponds to a list of  key-value pairs, where the first element is a   trie node hash and the second the rlp encoded data
      - `stopPath` - if the requested chunk is too large, it contains the path at which the server stopped building the chunk, used by the client to resume downloading from this path
    - `error` - a list of two elements, `code` and optional `data`, e.g. [`code`, `data`] - possible codes are
      - `0x0` no chunk for this root, but if `data` is present then it is the closes root for which it is available
      - `0x1` no chunk for this prefix
      - `0x2` back-off, too many requests
      - `0x4` invalid request

## Partially synced clients

Whit this mechanism, a partially synced client can serve other clients for the chunks it has available, provided that the chunks are fully validated.

NOTE:
Is this possible with the current fast and warp sync, and the proposed leaf, firehouse and red queen syncs? 

Also, as stated above, chunks need to be valid, meaning that this most likely wont work for leaf based sync, unless we can guarantee that they are valid. This is much easier with the full chunk.

## Light clients

A light client can be built on top of a protocol like this. For example, a client running in light mode, can sync only the chunks that it requires based on the accounts/storage keys that are requested from it, but just as in the case of partially synced clients, it can also serve other nodes that are syncing up by delivering only the chunks it has.

## Additional considerations

- Some prefixes might be empty, but provided that the trie is relatively balanced and the prefix is not too large, it should be a rare occurrence. It would be nice to be able to verify this though.
- Clients need to parse the chunk to extract the accounts. For one to validate and store the chunks, but also to extract all the storage roots and subsequently request those as well. Parsing the chunk (traversing), should not be very costly, but something not required when sending just the leafs.

## Prefix parity

Since keys and prefixes are sent as nibbles, some schema to signal parity is required. I'm sure there are tons of ways of doing it, but I haven't though about it too much yet, that said what's outlined in [Fire house sync](https://notes.ethereum.org/eXnqtO_vQquzrFDPHjuaFQ#Key-Prefixes), seems to be a good starting point.
