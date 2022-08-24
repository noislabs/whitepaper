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

[drand]: https://drand.love
[loe]: https://en.wikipedia.org/wiki/League_of_entropy
