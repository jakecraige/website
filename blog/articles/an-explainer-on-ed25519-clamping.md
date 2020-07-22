# An Explainer On Ed25519 Clamping

In Ed25519 signatures, there is a place where some "bit-twiddling" is done on the
private key which is typically called "clamping". The information on the
reasoning for this is sparse and the descriptions that do exist never went into
enough detail for me to fully understand it. This post aims to explain the
rationale for doing this clamping and why it works.

## What is clamping?

Clamping is the action of applying some deterministic manipulation to some
input bytes, typically using bitwise operations. It can be used for many
purposes, but one common use is to force arbitrary values into a particular
integer range by setting or zeroing bits in a particular way. For Ed25519 it's
used for other reasons which we will cover later, but is done on a 32-byte
input and looks like so:

```
input[0] &= 248
input[31] &= 127
input[31] |= 64
```

## When is clamping used?

Clamping is used to deterministically derive the private & public key from
a 32-byte "seed". This seed is provided as input to an implementation when you
want to derive the public key or sign a message. Here are where we see this done
in the golang standard library:

First, in key generation, it derives the public key using the following logic.
Ed25519 also requires that you SHA512 the seed and use the first 32-bytes for
the clamp, but it's not relevant to the clamping itself. That step is done
simply to protect against bad seed randomness which is non-uniform.

```
digest := sha512.Sum512(seed)
digest[0] &= 248
digest[31] &= 127
digest[31] |= 64

var A edwards25519.ExtendedGroupElement
var hBytes [32]byte
copy(hBytes[:], digest[:])
edwards25519.GeScalarMultBase(&A, &hBytes)
var publicKeyBytes [32]byte
A.ToBytes(&publicKeyBytes)

// source: https://github.com/golang/go/blob/11f92e9dae96939c2d784ae963fa7763c300660b/src/crypto/ed25519/ed25519.go#L98-L141
```

The second place is during signing, we see the same clamping is applied. In the
golang implementation the "private key" is a 64-byte value which is `seed ||
public_key`, so the second line is where the seed is being fed into the hash
function.

```
h := sha512.New()
h.Write(privateKey[:32])

var digest1, messageDigest, hramDigest [64]byte
var expandedSecretKey [32]byte
h.Sum(digest1[:0])
copy(expandedSecretKey[:], digest1[:])
expandedSecretKey[0] &= 248
expandedSecretKey[31] &= 63
expandedSecretKey[31] |= 64

// source: https://github.com/golang/go/blob/11f92e9dae96939c2d784ae963fa7763c300660b/src/crypto/ed25519/ed25519.go#L162-L171
```

While I've used the golang standard library as an example, you will be able to
find similar code in almost any library implementing ed25519. They are almost
all ports of the [SUPERCOP] implementation so the code is all very similar.

[SUPERCOP]: http://bench.cr.yp.to/supercop.html

## But why do we need to do this?

With the background out of the way, now we can get to the meat of this post.
What exactly is the clamping doing and why is it done?

The first thing to be aware of when we're dealing with byte arrays in ed25519
is they are represented in [little-endian] which means that the least
significant byte is first. This is important to keep in mind because if you
start thinking in big-endian this will be very confusing.

[little-endian]: https://en.wikipedia.org/wiki/Endianness

```
input[0] &= 248  // 248 == 0b11111000
input[31] &= 127 // 127 == 0b01111111
input[31] |= 64  // 64  == 0b01000000
```

In the first line, we are "clearing" or "zeroing" the lowest three bits of the
first byte. This works because if we `&` with 0, it will always be 0 in the
output, but if we `&` with 1, it will retain whatever value was already there.

The second line and third line work together to clear the 256th bit and set the
255th bit to 1. The second line does the clearing using the same logic as the
first and the third uses `|` to "set" a specific bit. It works using similar
logic to how `&` works, if we `|` with 0 we retain whatever value was there,
and if we `|` with 1 it will always produce 1, so we can set a specific bit.

At the end of this clamping we've taken a 32-byte value and cleared the lowest
3 bits and set the 255th bit as the highest bit.

### Clearing the lowest three bits

Cleaning these bits does something known as "clearing the cofactor". A cofactor
is one of the parameters that make up an elliptic curve. The number of points
on the elliptic curve can be described as `n = r * h` where r is the prime
order of a subgroup and h is the cofactor. If the cofactor is 1, like in many
of the standardized curves, it's not something we need to worry about.  If it
is not one, careful consideration needs to be made for it's implications in
cryptographic schemes to avoid a few types of attacks, notably [small-subgroup
attacks] and more [nuanced malleability attacks that affected
Monero][monero-attack].

As a consequence of Lagrange's Theorem, this means that the possible orders of
the curve's subgroups are the divisors of `n`.

> **Lagrange's Theorem**: For any finite group G, the order (number of
> elements) of every subgroup H of G divides the order of G.

In ed25519 we use [edwards25519] which has a prime `r` and cofactor of 8. This
means that set of divisors of `n` is the union of the divisors of `r` and 8.
Since `r` is prime it's divisors are simply `{1, r}`, the divisors of 8 are
`{1, 2, 4, 8}`. Putting all this together, every subgroup of edwards25519 must
have one of the following orders `{1, 2, 4, 8, r}`.

Back to the bits, what does clearing the lowest three do to the integer? It
turns it into multiple of 8, the cofactor. Here's why this is useful. In
protocols like diffie-hellman where you combine a secret with an externally
provided point, there is the possibility of [small-subgroup attacks] which can
cause you to leak your secret. By making the secret a multiple of 8, we no
longer have to worry about this and we will always get a point in the prime
order subgroup.

Let's assume that an adversary provides a point `H` from the subgroup with
order 8 and we multiply it by our secret `x` to get the diffie-hellman point `S
= xH`. The order of a group is defined as the smallest positive integer m such
that a^m = e, where e denotes the identity element of the group.

Because our secret is a multiple of 8, we can rewrite `S` as `S = xH = 8(mH) =
e`. Notice that this doesn't only work for subgroups of order 8, but for any
small subgroup because it can always be rewritten as a multiple of the order.
For example, if `H` has order 4, `S = xH = 8(mH) = 4(2mH) = 1` and we get the
same identity element. If the provided point is in the prime order `r`
subgroup, then we get some other point in the subgroup and _not_ the identity
element.

Because the identity element is in every subgroup (by definition of a group), we
can say that doing a scalar multiplication of this secret with an arbitrary
point from the group will result in an element in the prime order subgroup. In
practice, we may not actually want to allow the identity element and could
explicitly check this and reject the point if it was found, but this mitigates
the secret leakage that you would typically get which allows small-subgroup
attacks to be effective.

**So does this clearing matter for ed25519 signatures?** Surprisingly, no! We
are always doing scalar multiplication against a publicly known generator from
a subgroup that we know has prime order. We never multiply our secrets by
arbitrary points. The reason why this is done is so that the same secrets could
be also be used safely with [X25519] if you also need to do a key-exchange.

For compatibility with the standard, libraries, and to make sure it's still
safe to use with X25519 it's a good idea to do this if you can, but it's not
critical for the security of the signatures themself.

[small-subgroup attacks]: https://tools.ietf.org/html/rfc2785
[monero-attack]: https://web.getmonero.org/2017/05/17/disclosure-of-a-major-bug-in-cryptonote-based-currencies.html
[edwards25519]: https://tools.ietf.org/html/rfc7748#section-4.1
[X25519]: https://tools.ietf.org/html/rfc7748#section-5

### Setting the highest bit

Now we turn to the next part of clamping: clearing the 256th bit and setting the
255th to 1. The purpose of this is to ensure that the highest bit is always at
a fixed position. The first [explanation was seen on StackOverflow in
2013][high-bit] and later Daniel J. Bernstein (djb), the creator of ed25519,
explained in [a mailing list post from 2014]. I'll expand on these here.

X25519 only deals in x-coordinates and there is a simple & efficient way to
implement scalar multiplication of x-coordinates known as the [Montgomery
ladder][ladder]. The problem with this is that some implementations implement
it in variable-time based on the position of the highest bit. To avoid
implementations having to care about this he decided to make this part of the
standard so that if the implementation is variable time in that way, it will
run in constant time because of this clamping. There are alternative ways to
make this constant time covered in [Section 4.6.2][ladder], but this is what
was chosen for ed25519.

**So does this bit-setting matter for ed25519 signatures?** Again,
surprisingly, not really! In signatures we don't use the Montgomery Ladder
because there are more efficient ways to do scalar multiplication with a fixed
point, and we always multiply by the known generator. From [BL17][ladder]:

> Extensive research has produced a wide range of more complicated
> scalar-multiplication methods (for pointers see, e.g., [BDL+11],[BCLS14], and
> [BL16]), outperforming the Montgomery ladder for tasks such as computing n→nP for
> a fixed point P, or computing nth multiples of points on certain special curves, but
> the Montgomery ladder seems practically un-beatable for the core task of
> computing n, P→nP on typical curves.

An implementation _could_ use the Montgomery ladder with y-coordinate recovery
but standard implementations will not. As with clearing the bits, we see yet
another choice made to protect against bad implementations or using the same
keys with X25519.

If you want to get into the weeds learning about the Montgomery ladder
[BL17][ladder] and [CS17][Costello] are good resources.

[high-bit]: https://crypto.stackexchange.com/a/11818
[a mailing list post from 2014]: https://mailarchive.ietf.org/arch/msg/cfrg/pt2bt3fGQbNF8qdEcorp-rJSJrc/
[ladder]: https://eprint.iacr.org/2017/293.pdf
[Costello]: https://eprint.iacr.org/2017/212.pdf

## Wrapping up

The designs of Curve25519, Ed25519, and X25519 are filled with choices to
optimize for security and implementation simplicity to avoid many types of
failures that were seen in practice before its design. The need for clamping
and rationale definitely fits the bill but it has been hard to find a detailed
explanation on the reasoning for this and how necessary they are (or are not).
Hopefully this post helps with that :)

If you find any errors in this post or have any questions, you can [comment on Reddit][reddit] or
reach out to me on Twitter ([@jakecraige](https://twitter.com/jakecraige)).

[reddit]: https://www.reddit.com/r/crypto/comments/hvdzzy/an_explainer_on_ed25519_clamping/
