# Developer Friendly

## Noise Proxy Contract

Nois is meant to be a layer on top of which dapps can tap into. Therefore, its usage must be as simple as possible and not less.
In this spirit, our proxy contract have a main entry point:
```
getNextRandomness()
```
which will automatically calculate the next drand round and return the associated randomness via an IBC callback.
The developer does not have to think about when should he ask randomness nor how. We believe that this single API call 
will be able to fit most uses cases using randomness onchain.
The API is also offering more advanced entry points that need to be dealt with care for specific applications.

## Library

On top of this API, we offer a general toolbox that offers a handful of methods such as:
* `randFloat(seed)` that derives a random number between 0 and 1 from the randomness seed
* `range(seed, min, max)` that returns a random integer in the `[min,max(` range
* `shuffle(seed, list)` that shuffles a given list given the randomness seed

The team have seen many insecure implementations of these functions in the wild, and we believe it is important to provide
them as part of the Nois ecosystem to incentivize dapps developer to create secure and robust applications.
