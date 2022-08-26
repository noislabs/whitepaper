# Performance

## Commit to Callback Time (C2C Time)

Each Drand round is published at fixed points in time calculated as follows:

```
publish_time := genesis + (round - 1) * 30
```

where `genesis` is the drand network's start time (UNIX timestamp), `round` is an incremening integer starting at 1 and `30` is the 30 seconds round time.

Once this `publish_time` is reached, the randomness needs to be considered public. No matter if the chain or the contract knows the value already, any user can know the random value by observing the off-chain drand network. So it is important that after `publish_time` no actions for that round are allowed anymore. Think of it as closing submission of lottery tickets. Now the round should be processed as fast as possible in order to reveal the results and allow to continue the operation in case the next steps depend on it.

The time between beacon publishing an the callback consists of the following components:

- **Bot submission:** Multiple bots should observe the drand gossip network through varius communication protocols. Once a new round is found, they should craft a transaction, sign it and sent it to the Nois mempool. The fastest submission of one of the bots matters. When well connected, this should be doable in 1 second.
- **Block inclusion time:** When the beacon is in the mempool, the chain should ensure it is included in a block as fast as possible. With 5 second block times and an inclusion in the first or second block, this should be up to 10 seconds. The same transaction processes pending jobs and sends IBC messages. The block is executed in well under 1 second.
- **IBC relaying:** An IBC relayer picks up the message from the Nois chain and relays it to the destination chain. This depends on well configured and well connected relayers as well as availability of block space on the destination chain. In order to be included as fast as possible, the relayer should pay a transaction fee that is accepted by all validators. But in a high trafic sitiation, we don't get guearantees to be included quickly. So 5-20 seconds is a reasonable estimate.
- **Acknowledgement:** An IBC acknowledgement is sent to Nois but this is nothing the contract needs to wait for.

Once those steps are done, the callback is executed within 30 seconds of publishing. On a well configured network, average timings can be much faster though.

## Choice of round

The application needs to commit to a round number before the beacon is revealed.
Fortunately we have a relyable [BFT Time](https://docs.tendermint.com/master/spec/consensus/bft-time.html) but this is not perfectly accurate and can be behind. In case the contract thinks `publish_time` is not yet reached while the beacon is already published, an attacker can abuse the knowledge of the randomness. So we intoduce a duration `safety_margin` and require `publish_time` to be at least `safety_margin` after the current BFT time (`block_time`).

Using the formular from above, we want

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

This calculation can be generalized if an end time should be set in advance instead of closing right away. With flexible end times, the publish times of beacons can be matched and only the safety margin need to be considered.

## Short Block Times

The Nois network can consider to reduce block times from the typical 5-7 seconds in Cosmos to something shorter. Doing so has to be carefully tested in environments with many globally distributed validators. But seeing teams successfully testing 1s block times is promising.

## High Frequency Beacons

Right now drand emits a new beacon every 30 seconds. But there are plans to reduce that round time to something like 3 or 5 seconds. This will allow Nois to operate on higher frequency.

## Summary

When comitting to a round in 2-32 seconds that needs roughly 10-30 seconds to be processed, you get a delay between commitment to callback of 12-62 seconds. Various usecase specific optimization techniques are to be explored.

This end-to-end randomness distribution speed can improve significantly with the upcoming drand frequency change.
