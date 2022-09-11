# Further Work

The solution above explains what we can do with technology available today. But we'll not stop there. The next steps might be the following (unsorted).

## IBC Queries

We'll closely follow the development of IBC queries, which is a technology we assume to allow for faster round trips and lower fees.

## Alternative Entropy Sources

We love what drand and the League of Entropy brought to the internet, which is nothing less than the first decentralized entropy source. But choice is important. Some applications may wish to sacrifice some of the security properties in order to get faster and cheaper randomness. Some users might demand governance voting for drand [MPC] participants. The options are to be explored, but the network will be designed for multiple sources.

## Internalizing randomness generation

The node operators of the Nois Network could integrate randomness generation into their own operations. There has been work to [integrate random beacons into Tendermint](https://medium.com/@dgaminghub/arcade-tendermint-hack-with-built-in-threshold-bls-random-beacon-for-applications-a51eafb77f53) that could be explored. But also a second process that runs along with the validator node would be an option.

## More features

Drand is releasing a timed encryption feature that is using the drand randomness
as a basis. Integrating that with Nois Network would make sense and offer the
first timed encryption feature on the Cosmos ecosystem.

[mpc]: https://en.wikipedia.org/wiki/Secure_multi-party_computation
