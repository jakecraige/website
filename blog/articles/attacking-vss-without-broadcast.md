# Attacking VSS Without Broadcast

[Verifiable Secret Sharing (VSS)][] is a method of secret sharing where extra information is
included alongside the secret shares which allows each player to verify the share they received is
consistent with the others. In other words, the share is valid and can be combined with others to
deterministically reconstruct the shared secret.

VSS is commonly used in multi-party computation protocols to perform a distributed key generation
(DKG) where no party learns the secret but they can still perform computations with it. These are of
particular interest now as the blockchain space has taken a lot of interest in using threshold
signatures to secure private keys and starting to deploy these in practice.

One key assumption made for the security of VSS is the existence and usage of a "broadcast channel"
that is used to share the proof with all parties. A broadcast channel is a communication channel
which any party can send a message on, and every other party is able to receive the message in an
unmodified form.

This post describes an attack that can be used to forge VSS proofs _when the broadcast channel is
not present_, and then utilizes it to break a threshold signing scheme.

## The DKG

We will be using Shamir's Secret Sharing with the [Pedersen VSS][] as our DKG. This DKG uses
parallel executions of the [Feldman VSS][] to enable us to have t participants create a random value
of which t-of-n can reconstruct the secret.

## Forging a proof

Our adversary is tasked with creating a proof for some provided $y_0$ where $y_0 = g^{a_0}$ and
$a_0$ is not known. At a high level, the attack works by generating a separate proof for each
party, where we perform a "key-cancellation" or "rogue key" strategy to construct a valid proof. We
can see how the broadcast channel prevents this attack by guaranteeing all parties receive the same
proof. The adversary constructs the proofs as follows:

1. A sharing polynomial with random coefficients is created: $p(x) = \sum_{i=2}^{t-1} a_i x^i$.
1. Shares for each party are created as normal: $\{s_i = p(i) | i \in [n]\}$
1. Commitments to each coefficient are created as normal: $\{y_i | g^a_i \in [2, t-1]\}$
1. For each participant $j \in [n]$ a separate $y_1$ used in each proof:
  1. $y_{1,j} = (y_0^{-1})^{j^{-1}}$
  1. $\{y\_0, y\_{1,j}, y\_2, ..., y\_{t-1}\}$

Verification of the VSS for each participant $j$ is defined as
$g^{s\_j} = \prod\_{i=0}^{t-1} {y\_i^j}^i$. We show that the proof holds below:

$$
\begin{align\*}
g^{s\_j} &= g^{a\_0 j^0} \cdot g^{a\_1 j^1} \cdot \prod\_{i=2}^{t-1} g^{a\_i j^i} \\\\
\&= g^{a\_0 j^0} g^{(-a_0 j^{-1}) {j^1}} \prod\_{i=2}^{t-1} g^{a\_i j^i} \\\\
\&= g^{a\_0 j^0 + (-a\_0 j^{-1}) {j^1}  + \sum\_{i=2}^{t-1} a\_i j^i} \\\\
\&= g^{a\_0  - a\_0 + \sum\_{i=2}^{t-1} a\_i j^i} \\\\
\&= g^{\sum\_{i=2}^{t-1} a\_i j^i} \\\\
\end{align\*}
$$

## The Threshold Signature scheme

A standard Schnorr threshold signature scheme that generates nonces deterministically.

More detailed description coming soon.

## Attacking the scheme

The attacker requests a signature for same message and provides the same shares, but changes the
public nonce it commits to using the above attack. This thus changes the challenge value and creates
a standard nonce reuse problem.

More detailed description coming soon.

[Pedersen VSS]: https://link.springer.com/content/pdf/10.1007/3-540-46416-6_47.pdf
[Feldman VSS]: https://www.cs.umd.edu/~gasarch/TOPICS/secretsharing/feldmanVSS.pdf
[Verifiable Secret Sharing (VSS)]: https://en.wikipedia.org/wiki/Verifiable_secret_sharing


<script>
  MathJax = {tex: { inlineMath: [['$', '$'], ['\\(', '\\)']] }, svg: { fontCache: 'global' }};
</script>
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
