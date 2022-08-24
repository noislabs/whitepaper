# Our Approach

Nois Network aims to provide a safe and secure solution to the problem that is native to the IBC world and provides the best possible user experience for a wide range of applications.

In contrast to other consensus algorithms, Tendermint-based blockchains do not need randomness at block production layer. As a consequence no random value can trivially be passed to applications from consensus. So our randomness solition will live entirely on the app level, allowing us to keep block production going even when there is a hickup in the the randomness part.

In the first iteration, Nois will use random beacons produced by the [drand] network, which is powered by a consortium of participants that generate randomness using multi-party computation. This is implemented using BLS threshold signatures, which produces unpredictable values that cannot be manipulated by any of the drand participants. The drand mainnet is instantiated by the [Legue of Entropy][loe], which has been operating it in production for more than two years. For example, Filecoin relies on drand for block production.

Drand random beacons can be submitted to blockchains that perform BLS signature verification. This way we can build a random oracle that securely brings randomness on chain. This method was [described and proven in 2020 for CosmWasm](https://medium.com/@simonwarta/when-your-blockchain-needs-to-roll-the-dice-ed9da121f590). A few months later, this proof of concept was turned into production by [Terrand](https://docs.terrand.dev/). BLS verification is a computationally heavy operation, but leveraging the strength of the Rust optimizer and Wasm's near native execution speed, Drand beacons could be verified for less than 3$ in gas fees on Terra.

The next step in the evolution is to make drand beacons easily accessible by as many dapps as possible in a way that is easy to use and affordable. In an ideal world a dapp developer would just do something like this:

```rust
// pseudo-code
let beacon: [u8; 32] = await getRandom();
```

We believe the burden of implementing drand verification once per contract or even once per blockchain is too much in an ecosystem that is preparing for thousands of independent and inter-connected blockchains. So instead of executing the drand verification on the chain of the dapp, IBC is used to request the beacon from the Nois Network.

### How it works

The following steps are taken to get the randomness:

1. Contract on a CosmWasm-enabled chain (such as Juno or Tgrade) sends a message to a Nois proxy contract on the same chain. A reply with further information regarding the job is sent to the original contract.
2. The proxy contract sends an IBC message to its couter-part on the Nois Network where the job is put in the queue.
3. Once the drand beacon of the correct round is released, a network of bots sends it to the Nois Network for verification.
4. After successful verfication, the pending jobs for the round are processed. For every matching job, an IBC response with the beacon is sent.
5. The proxy contract receives the beacon and sends a callback to the original contract.

### Commit to Callback Time (C2C Time)

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

#### Choice of round

The application needs to commit to a round number before the beacon is revealed.
Fortunately we have a relyable [BFT Time] but this is not perfectly accurate and can be behind. In case the contract thinks `publish_time` is not yet reached while the beacon is already published, an attacker can abuse the knowledge of the randomness. So we intoduce a duration `safety_margin` and require `publish_time` to be at least `safety_margin` after the current BFT time (`block_time`).

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

#### Summary

When comitting to a round in 2-32 seconds that needs roughly 10-30 seconds to be processed, you get a delay between commitment to callback of 12-62 seconds. Varous usecase specific optimization techniques are to be explored.

[bft time]: https://docs.tendermint.com/master/spec/consensus/bft-time.html
[drand]: https://drand.love
[loe]: https://en.wikipedia.org/wiki/League_of_entropy
