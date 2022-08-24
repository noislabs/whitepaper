# Further Work

The solution above explains what we can do with technology available today. But we'll not stop there. The next steps might be the following (unsorted).

## High Frequency Beacons

Right now drand emits a new beacon every 30 seconds. But there are plans to reduce that round time to something like 3 or 5 seconds. This will allow Nois to operate on higher frequency.

## Short Block Times

The Nois network can consider to reduce block times from the typical 5-7 seconds in Cosmos to something shorter. Doing so has to be carefully tested in environments with many globally distributed validators. But seeing teams successfully testing 1s block times is promising.

## IBC Queries

We'll closely follow the development of IBC queries, which is a technology we assume to allow for faster round trips and lower fees.

## Alternative Entropy Sources

We love what drand and the League of Entropy brought to the internet, which is nothing less than the first decentralized entropy source. But choice is important. Some applications may wish to sacrifice some of the security properties in order to get faster and cheaper randomness. Some users might demand governance voting for drand [MPC] participants. The options are to be explored but the network will be designed for multiple sources.

## Internalizing randomness generation

The node operators of the Nois Network could integrate randomness generation into their own operations. There has been work to [integrate random beacons into Tendermint](https://medium.com/@dgaminghub/arcade-tendermint-hack-with-built-in-threshold-bls-random-beacon-for-applications-a51eafb77f53) that could be explored. But also a second process that runs along with the validator node would be an option.

[mpc]: https://en.wikipedia.org/wiki/Secure_multi-party_computation
