---
layout: post
title: Rotating CKKS Ciphertexts
date: 2023-01-24 23:50 +0100
list: false
---
This post explains how ciphertext slots can be rotated in the CKKS FHE cryptosystem.
Upon spending some time with the FHE community, I noticed that some people asked for explanations on how ciphertext rotations work in the CKKS cryptosystem.
The purpose of this post is to provide the reader with an intuitive and practical understanding of CKKS ciphertext rotation as well as an understanding of the terminology to comprehend the literature.

{% raw %}
$$
   \def\z{{\zeta}}
   \def\Q{{\mathbb{Q}}}
   \def\R{{\mathcal{R}}}
   \def\Z{{\mathbb{Z}}}
   \def\C{{\mathbb{C}}}
$$

## Background

First, let us examine some mathematical background and the encoding used by CKKS.
We will use the same notation as the original CKKS publication.

Given a pair of fields such that $E \subseteq F$ and that the operations of $E$ are those of $F$ restricted to $E$, then $F$ is a **field extension** of $E$.
We will denote this as $F/E$.

If every element of $F$ is the root of a non-zero polynomial with coefficients in $E$, then such a field extension is known as **algebraic**.

An **automorphism** of $F/E$ is an isomorphism $f: F \rightarrow F$ such that $f(x) = x$ for all $x \in E$.
$Aut(F/E)$ denotes the group of automorphisms of $F/E$ with the group operation being composition.

If $\\{x \in F | f(x) = x, \forall f \in Aut(F/E) \\} = E$ and $F/E$ is algebraic, then $Aut(F/E)$ is the **Galois group** of $F/E$.
We will denote the Galois group of $F/E$ with $Gal(F/E)$.


$\Phi_M(X)$ denotes the $M$-th cyclotomic polynomial of degree $N = \varphi(M)$.
Furthermore, we define $\z(M) := exp(- 2\pi i / M)$ and $\R := \Z[X]/ (\Phi_M(X))$.
We denote the multiplicative group of units in $\Z_M$ as $\Z^\*_M := {x \in \Z_m : gcd(x, M) = 1}$.

The authors of CKKS state that it is known that $Gal(\Q(\z(M))/ \Q)$ consists of the mappings $\kappa_k: m(X) \mapsto m(X^k) \mod (\Phi_M(X))$ for all $k$ co-prime with $M$ and $m(X) \in \R$.
This group is also isomorphic to $\Z^*_M$.

The authors also explain that applying a Galois element $\kappa_j$ to the entries of a ciphertext under the key $s$ results in a ciphertext that encrypts a permutation of the slots under the key $\kappa_j(s)$.
Practically, key switching can be used to obtain a ciphertext of the same message under $s$ again.

We can easily see how it is possible to apply a permutation to a ciphertext.
But it is still not clear how the elements of the Galois group can be used to realise rotations.

## CKKS Decoding and Encoding

Explaining the jump from permutations to rotations requires a slight detour to the decoding and encoding of CKKS.
Decoding turns a message polynomial $\in \R$ into a vector of complex numbers $\in \C^{N/2}$ whereas encoding does the opposite.

The first step of decoding removes the scaling factor $\Delta$.

For the second step, decoding takes the result $ \Delta^{-1} m(X)$ and evaluates it at all the roots of $\Phi_M(X)$.
These roots generally have the form $\z(M)^j$, where $j$ is smaller than and coprime with $M$.
If $M$ is a power of two, then the $M/2$ roots would be: $\z(M), \z(M)^3, \z(M)^5, ..., \z(M)^{M-1}$.

Finally, we have to apply a projection which removes half of the complex numbers.
Understanding the reasoning behind this projection requires a small detour.

We use Euler's formula $e^{ix} = cos(x) + i \cdot sin(x)$ to represent the roots in a unit circle.
The x axis corresponds to the real part and the y axis to the imaginary part of the complex number.

![Euler's formula depicted graphically.](/assets/0123/eulers_formula.svg)

The $M/2$ roots can be grouped into pairs $(\z(M)^k, \z(M)^{M-k})$ such that $\z(M)^k = \overline{\z(M)^{M-k}}$.
On the unit circle, this corresponds to mirroring a root along the x axis.

![Depiction of the roots of the 8th cyclotomic polynomial on a unity circle.](/assets/0123/ckks_roots.svg)

We use a projection so that we only retain the complex number of one root of each pair since the other number will simply be its conjugate and can thus not be freely chosen during the encoding.
Let $T$ be a subgroup of the multiplicative group $\Z^{\*}_M$ such that $\Z^{\*}_M / T = \\{-1, 1\\}$, then T contains the indices that are preserved by the projection.
This results in $M/4$ complex numbers which form the decoded message.

Encoding performs the inverse of this operation, the only major difference is that rounding is required to map the polynomial coefficients to integers.

## Rotating

In order to understand how rotation can be achieved, let us examine the Microsoft SEAL implementation of CKKS.
The function `void Evaluator::apply_galois_inplace` applies a Galois element to a ciphertext and then proceeds to perform a keyswitch.
These keyswitches require a so called GaloisKey.
This is just a keyswitching key for a specific Galois element, i.e. it switches from the key $\kappa_k(s)$ to $s$, where $\kappa_k$ is the Galois element.

Generating a GaloisKey for every Galois element would consume a lot of space, but elements can be applied successively to obtain a new element, for example: $\kappa_3 \circ \kappa_9 = \kappa_{27}$.
This allows to reduce the size of the GaloisKeys and the time required to generate them in exchange for increased noise and runtime for applying a Galois element.

Finally, we can talk about rotations themselves.
Let us assume that $M$ is a power of two and that we are provided with a ciphertext $ct(X)$.
If we now apply the Galois element $\kappa_3$ to $ct(X)$, then we obtain $ct(X^3)$.
The same morphism is also applied to the contained message, which is now $m(X^3)$.
This should correspond to a rotation by a single slot.

If we evaluate $m(X^3)$ with $X = \z(M)$, then we obtain the same result as if we had evaluated $m(X)$ with $X = \z(M)^3$.
So this slot has been successfully switched, but what about the next one?
$m(X^3)$ at $\z(M)^3$ corresponds to $m(X)$ at $\z(M)^9$ rather than $\z(M)^5$...
However, we are not even aware if both 3 and 5 are $\in T$.

So let us take a closer look at how we might construct $T$.
Ben Lynn's notes (see External Links) provide some results that are very important for this.
In particular, if $a \equiv \pm 3 \mod 8$, then the order of $a \mod 2^t$ is $2^{t-2}$, otherwise it is strictly smaller.
We may define $T$ as the cyclic subgroup of $\Z^{\*}_M$ generated by $3$.
In that case, the third slot would also correspond to the evaluation at $\z(M)^9$ and the previous problem disappears.
So we obtain the decoded message as: $(\Delta^{-1}M(\z(M)), \Delta^{-1}M(\z^3(M)), \Delta^{-1}M(\z(M)^{3^2 \mod M}), \Delta^{-1}M(\z(M)^{3^3 \mod M}), ..., \Delta^{-1}m(\z(M)^{3^{M/4 - 1} \mod M}))$.

### External Links
Wikipedia on field extensions: <https://en.wikipedia.org/wiki/Field_extension>

Wikipedia on Galois groups: <https://en.wikipedia.org/wiki/Galois_group>

Ben Lynn's notes on number theory: <https://crypto.stanford.edu/pbc/notes/numbertheory/gengen.html>

Original CKKS paper: <https://link.springer.com/chapter/10.1007/978-3-319-70694-8_15>

Microsoft SEAL: <https://github.com/microsoft/SEAL>

SEAL applying a Galois element: <https://github.com/microsoft/SEAL/blob/4.1.1/native/src/seal/evaluator.cpp#L2362>

SEAL calculating the mapping from coefficient to slot: <https://github.com/microsoft/SEAL/blob/4.1.1/native/src/seal/batchencoder.cpp#L64>

{% endraw %}
