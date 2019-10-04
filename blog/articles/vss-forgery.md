# VSS Forgery Against Bad Broadcast Channels

[Verifiable Secret Sharing (VSS)][] is a method of secret sharing where extra information is
included alongside the secret shares that allows each player to verify the share they received is
consistent with the others. In other words, the share is valid and can be combined with others to
deterministically reconstruct the shared secret.

VSS is commonly used in multi-party computation protocols to perform a distributed key generation
(DKG) where no party learns the secret but they can still perform computations with it. These are of
particular interest now as the blockchain space has taken a lot of interest in using threshold
signatures to secure private keys and starting to deploy these in practice.

One key assumption made for the security of VSS is the existence and usage of a "broadcast channel"
to share the proof with all parties. A broadcast channel is a communication channel which any party
can send a message on, and every other party is able to receive the message in an unmodified form.

This post describes an attack that allows an adversary to forge VSS proofs for any public key without
knowledge of the discrete log. It only works in systems where broadcast channels are not implemented
correctly.

## Forging the proof

We will use Shamir's Secret Sharing with the [Feldman VSS][] for this forgery.

Our adversary is tasked with creating a proof for some provided $y_0$ where $y_0 = g^{a_0}$ and
$a_0$ is not known. At a high level, the attack works by generating a separate proof for each
participant, where we perform a "key-cancellation" or "rogue key attack" to make the proof valid for
each participant.

The adversary constructs the proofs as follows:

1. A sharing polynomial with random coefficients is created, $p(x) = \sum_{i=2}^{t-1} a_i x^i$, but
   we do not include the first two coefficients as we normally would.
1. Shares for each participant are created: $\\{s_i = p(i) | i \in [n]\\}$
1. Commitments to each coefficient are created: $\\{y_i | g^a_i \in [2, t-1]\\}$
1. For each participant $j \in [n]$ a separate $y_1$ is created for each proof:
  1. $y_{1,j} = (y_0^{-1})^{j^{-1}}$
  1. $\\{y\_0, y\_{1,j}, y\_2, ..., y\_{t-1}\\}$

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

The resulting set of proofs convince the participants that their shares are legitimate shares of the
discrete log of $y\_0$, even though the adversary doesn't know it. Furthermore the reconstructed
shares will result in a value of zero (because there is no free coefficient in the polynomial) which
may lead to some interesting possible attacks when they are combined with other shares.

It's easy to see why this does not work when a broadcast channel is in place, because each proof is
specific to a participant, all but that participant would reject the proof as invalid.

While this attack only works against flawed implementations, this seems to be an interesting result
which may be useful in other circumstances.

[Pedersen VSS]: https://link.springer.com/content/pdf/10.1007/3-540-46416-6_47.pdf
[Feldman VSS]: https://www.cs.umd.edu/~gasarch/TOPICS/secretsharing/feldmanVSS.pdf
[Verifiable Secret Sharing (VSS)]: https://en.wikipedia.org/wiki/Verifiable_secret_sharing

<script>
  MathJax = {tex: { inlineMath: [['$', '$'], ['\\(', '\\)']] }, svg: { fontCache: 'global' }};
</script>
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
