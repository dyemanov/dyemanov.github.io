---
layout: post
title: Generic thoughts about the network performance
---

Everybody knows that the performance of the data transfer in the Internet world mostly depends not on the network bandwidth but on its latency (response times). In other words, the chattier is the protocol the worse is the overall performance. Transferring the same data as a large single packet will be faster than splitting it across multiple smaller packets.

At the software level, there may be different solutions attempted to minimize the number of network round-trips and thus improve the transmission times. Here you can see what has been done in Firebird up to date.

- While replying to the fetch request, the server batches as many records as it can fit into a single protocol packet. They are then cached on the client side and returned to the caller one by one. Once the local cache is exhausted, a new fetch request is sent to the server and the batching continues until the end-of-stream marker has been processed for the cursor. In order to optimize the process a little, the server side maintains its own cache of records which gets filled in the meantime, while waiting for the next fetch request from the client. *[Done many years ago during the InterBase era.]*
- Optimize the binary layout of the transmitted data, i.e. pack/compress them. In memory, all packets are represented in the expanded (structured) form but they are serialized accordingly to the [XDR](http://en.wikipedia.org/wiki/External_Data_Representation) rules before transmission. However, XDR is not a compression, its primary goal is cross-platform interoperability. So some packet types could benefit from the additional packing of their internals. For example, the server truncates the VARCHAR values up to their real length instead of sending them with the declared length. As a result, more records can fit a single protocol packet, thus reducing the number of round-trips (see above). *[Performed unconditionally starting with Firebird 1.5, more advanced packing techniques are possible.]*
- Batch multiple different packets into the single one. This can be done if the order of packets is deterministic and can be foreseen. Currently the processing of some packet types is deferred in order to be batched together with the next packet. For example, statement allocation requests are batched together with the first usage of those statement handles, releasing of statement and blob handles is delayed until the next packet is sent, execution of SELECT statements is deferred until the first fetch is attempted, etc. *[Done in Firebird 2.1, a few other optimizations of that kind are possible.]*

I'm going to address these items in more details in the subsequent blog posts.
