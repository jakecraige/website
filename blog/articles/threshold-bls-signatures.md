# Threshold BLS Signatures

It's' often said how easy it is do to BLS threshold signatures, but most
information you find when searching leads you to n-of-n multi-signatures \[[1]\]
\[[2]\], not threshold signatures. The few results that do mention they are
possible but don't provide technical details. This goal of this post is to
remedy that.

[1]: https://crypto.stanford.edu/~dabo/pubs/papers/BLSmultisig.html
[2]: https://eprint.iacr.org/2018/483

A **threshold signature** is a way to create a cryptographic signature from
a shared private key that uses a distributed set of participants. We typically
describe the parameters such that $t$ of $n$ (e.g.  3-of-5) are required to
create a signature. The security is defined that any less than $t$ participants
learn nothing about the shared private key and cannot produce a signature.

By distributing the signing across multiple locations, we make attacking the
system more difficult because a threshold of participants must be compromised to
recover the shared private key.

For more background on threshold signatures in general and BLS read [Threshold
Signatures Explained][binance-article] and [BLS Signatures for Busy
People][bls-busy].

Let's get into to the technical details. We focus on BLS12-381 in this post, but
the ideas are applicable to other curves too.

[binance-article]: https://www.binance.vision/security/threshold-signatures-explained
[bls-busy]: https://gist.github.com/hermanjunge/3308fbd3627033fc8d8ca0dd50809844

## Threshold BLS12-381

The first step is to generate the shared private key and create shares of it.
There are two ways we can approach this: trusted or distributed. I'll describe
a trusted version here, to make a distributed version you would use a protocol
like [GJKR06].

A BLS12-381 private key is a random value within the field $\mathbb{F}\_r$ where
$r$ is `0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001`
[\[source\]][bls12381-curve].

We use [Shamir's secret sharing][SSS] to split the private key into shares where
a threshold of them can reconstruct the private key. When creating shares, we
must use the $\mathbb{F}\_r$ field.

> **Implementation note**: _Shamir's secret sharing libraries are commonly
> implemented over the binary field $2^8$ and do not support providing a custom
> finite field. You will need to find a library that does or implement it
> yourself._

**Keygen**

1. Generate a random private key $k \leftarrow_{$} \mathbb{F}\_r$.
1. Split into $n$ shares $\set{k\_0, k\_1, \dots, k\_n} = \text{SSS}(k, t, n)$
   using Shamir's secret sharing over $\mathbb{F}\_r$.
1. Securely distribute the shares to participants.

[bls12381-curve]: https://electriccoin.co/blog/new-snark-curve/
[SSS]: https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing
[GJKR06]: https://www.semanticscholar.org/paper/Secure-Distributed-Key-Generation-for-Discrete-Log-Gennaro-Jarecki/bf9e630c13f570e2df05b6dcce3ea987015af7c3

**Signing**

Signing works the same as normal BLS signatures, but instead of using the
private key you sign using the share of the private key.

1. Each participant $i$ generates a partial signature $s\_i = H(m) \cdot {k\_i}$.

**Signature Aggregation**

After $t$ signatures have been collected, they can be aggregated into
a signature under the private key $k$. This is done by doing the Shamir
reconstruction on the partial signatures themselves.

Typically when we think of a Shamir reconsruction we are in a finite field of
integers $\mathbb{Z}\_q$, but with BLS the signature is a point within a group
$\mathbb{G}\_2$ instead of an integer. To do the reconstruction, we use the
standard [Lagrange interpolation], but do part in $\mathbb{F}\_r$ and the other
in $\mathbb{G}\_2$.

$$
\begin{align\*}
s &= \sum\_{i=0}^{t-1} s\_i \prod\_{j=0, j \neq i}^{t-1} \frac{x\_j}{x\_j - x\_i}\\\\
  &= \sum\_{i=0}^{t-1} (H(m) \cdot k\_i) \prod\_{j=0, j \neq i}^{t-1} \frac{x\_j}{x\_j - x\_i}\\\\
  &= H(m) \cdot \sum\_{i=0}^{t-1} k\_i \cdot \prod\_{j=0, j \neq i}^{t-1} \frac{x\_j}{x\_j - x\_i}\\\\
  &= H(m) \cdot k
\end{align\*}
$$

> **Implementation note**: _The Lagrange basis (the product component of the
> first equation), is computed in $\mathbb{F}\_r$ and it is then multiplied with
> the $s\_i$ using point multiplication in $\mathbb{G}\_2$ and the result is
> summed with point addition in the same group._
>
> _Similar to Shamir's secret sharing implementations, libraries don't often
> expose the APIs necessary to do this, so you'll need to find one that does or
> implement it._

The resulting $s$ value is a valid signature under the private key $k$.

[Lagrange interpolation]: https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing#Computationally_efficient_approach

---

In summary, we can see that threshold BLS signatures are a pretty
straightforward combination of secret sharing and group operations, there's just
a few tricky parts with making sure computations happen within the right group.

That's all folks! Hopefully this helps others researching this topic.

If you have questions or feedback, please reach out on Twitter
[@jakecraige](https://twitter.com/jakecraige) and I'm happy to discuss more.


