# Performance

## Commit to Callback Time (C2C Time)

Each Drand round is published at fixed points in time, calculated as follows:

```
publish_time := genesis + (round - 1) * roundTime
```

where `genesis` is the drand network's start time (UNIX timestamp), `round` is an incrementing integer starting at 1 and `roundTime` is the drand round time in seconds. At the beginning, `roundTime` is 30s but the drand project is planning to release a new network with much higher frequency (3-6s).

Once this `publish_time` is reached, the randomness needs to be considered public. No matter if the chain or the contract knows the value already, any user can know the random value by observing the off-chain drand network. So it is important that after `publish_time` no actions for that round are allowed anymore. Think of it as closing submission of lottery tickets. Now the round should be processed as fast as possible in order to reveal the results and allow to continue the operation in case the next steps depend on it.

The time between beacon publishing an the callback consists of the following components:

- **Nois bots submission:** bots should observe the drand gossip network through various communication protocols. Once a new round is found, they should craft a transaction, sign it and sent it to the Nois mempool. Note that only the fastest submission of one of the bots matters. When well-connected, this should be doable in under 1 second. This layer is completely permissionless, i.e. anyone can participate. Registered bot operators get compensated for this work.
- **Block inclusion time:** When the beacon is in the mempool, the chain should ensure it is included in a block as fast as possible. With 5-second block times and an inclusion in the first or second block, this should be up to 10 seconds. The same transaction processes pending jobs and sends IBC messages. The block is executed in well under 1 second.
- **IBC relaying:** An IBC relayer picks up the message from the Nois chain and relays it to the destination chain. This depends on well configured and well-connected relayers as well as availability of block space on the destination chain. In order to be included as fast as possible, the relayer should pay a transaction fee that is accepted by all validators. But in a high traffic situation, we don't get guarantees to be included quickly. So 5-20 seconds is a reasonable estimate.
- **Acknowledgement:** An IBC acknowledgement is sent to Nois, but this is nothing the contract needs to wait for.

Once those steps are done, the callback is executed within 30 seconds of publishing. On a well configured network, and depending on the block time, average timings can be much faster though.

## Choice of round

<!---
XXX Do we really need this ? I really dont get why we need a safety margin: at the point in time where the dapp request noise.getNextRandomness() then at this point, regardless of how th request is handled, the randomness is gonna come from a future round, so the app has nothing to worry about.
What I am missing ?


XXX Simon, Rewrite
-->

The application needs to commit to a round number before the beacon is revealed.
Fortunately we have a reliable [BFT Time](https://docs.tendermint.com/master/spec/consensus/bft-time.html) but this is not perfectly accurate and can be behind. In case the contract thinks `publish_time` is not yet reached while the beacon is already published, an attacker can abuse the knowledge of the randomness. So we introduce a duration `safety_margin` and require `publish_time` to be at least `safety_margin` after the current BFT time (`block_time`).

Using the formula from above, we want

```
(1) publish_time := genesis + (round - 1) * 30
(2) publish_time >= block_time + safety_margin

genesis + (round - 1) * 30 >= block_time + safety_margin
(round - 1) * 30 >= block_time + safety_margin - genesis
round - 1 >= (block_time + safety_margin - genesis) / 30
round >= ((block_time + safety_margin - genesis) / 30) + 1
```

This inequality is satisfied when doing

```
round := ceil((block_time + safety_margin - genesis) / 30) + 1
```

Assuming `safety_margin` is set generously to 2 seconds, the `round` calculated that way is 2-32 seconds in the future.

This calculation can be generalized if an end time should be set in advance instead of closing right away. With flexible end times, the publishing times of beacons can be matched, and only the safety margin need to be considered.

## Short Block Times

The Nois network can consider reducing block times from the typical 5-7 seconds in Cosmos to something shorter. Doing so has to be carefully tested in environments with many globally distributed validators. Fortunately, there has been teams successfully testing 1s block times and thus we believe it's a viable path forward ([1](https://twitter.com/fekunze/status/1542490680446050304), [2](https://twitter.com/crypto25807202/status/1551197364529967104), [3](https://docs.seinetwork.io/introduction/overview)).

## Process all drand rounds

The time between the publication of a random beacon and when it becomes available on chain is
important for the user experience. Depending on product design, this may be the time an end user
is staring at a spinner waiting for a result. For a great UX it is crucial to have a fast
bots that are well-connected to drand nodes using various transports (HTTP, pubsub, gRPC) and
submit a transaction to the Nois chain containing the beacon immediately when they first see it.

If the bots had to scan the Nois chain or even various customer chains to check if a round
was requested, valuable time is lost. A beacon request might already be in a mempool but is
not yet committed to a block. Or it is in a block but the block's events are not yet indexed.

By processing all drand round on the Nois chain, we remove communication overhead
and speed up processing of each round. At the same time, we optimize the chain for
drand verification, ensuring this does not lead to performance or storage issues.
