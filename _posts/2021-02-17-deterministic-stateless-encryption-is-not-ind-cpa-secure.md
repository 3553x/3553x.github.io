---
layout: post
title: IND-CPA Security & Deterministic Stateless Encryption
date: 2021-02-17 18:47 +0100
---
Encryption algorithms which for a given key always map the same message to the same ciphertext (i.e. deterministic, stateless algorithms) are not IND-CPA secure.
This is typically taught in introductory level courses on cryptography, however I noticed that some students seem to be confused about this and every now and then a question about this pops up on online discussion forums.
Hence, I have written this blogpost in hopes of clearing things up.

## IND-CPA

IND-CPA refers to ciphertext indistinguishability under a chosen-plaintext attack.
This property is defined in a game context: an attacker attempts to demonstrate that he can break the encryption scheme with a non-negligible advantage without violating the rules of the game.
Before we explain the details of the above sentence, let us define a game:

These games are defined with a security parameter \\(n\\) in mind.
For our purposes, this \\(n\\) would likely be the key size for the encryption scheme that we are using in our game.

$$
\begin{split}
Experiment\ & Exp_{\mathcal{SE}}^{ind-cpa}(A) \\
	& b \leftarrow^$ \{0, 1\} \\
	& K \leftarrow^$ K \\
	& b' \leftarrow^$ A^{E_k(LR(\cdot,\cdot,b))} \\
	& \text{if}\ b = b'\ \text{return}\ 1\ \text{else return}\ 0\\
\\
Oracle\ & E_k(LR(M_0, M_1, b)) \\
	& \text{return}\ E_k(M_b)
\end{split}
$$

\\(\leftarrow^$\\) indicates random sampling.
\\(\mathcal{SE}\\) is the symmetric encryption scheme and \\(E_k(M)\\) represents the encryption of the message \\(M\\) scheme with key \\(k\\).
Here an adversary \\(A\\) attempts to recover the value of \\(b\\) such that the experiment returns 1.
This indicates that an attacker can tell whether a ciphertext corresponds to a specific message.
Hence the name: ciphertext indistinguishability.
The adversary is given access to the encryption oracle.
However, he may merely specify the two arguments \\(M_0\\) and \\(M_1\\), which have to be of the same length.
He can not modify \\(b\\) and is (unless he is able to recover it from the ciphertexts with a successful attack) unaware of the value of \\(b\\) that is passed to the oracle.
\\(A\\) is also limited to a polynomial running time.
The adversary may guess of course, in which case he is going to be correct in about half of all cases.
The advantage is therefore defined to be:

$$
Adv_{\mathcal{SE}}^{ind-cpa}(A) = | 2  Pr[Exp_{\mathcal{SE}}^{ind-cpa}(A) = 1] - 1 |
$$

This slightly unintuitive definition stems from the fact that an attacker which never guesses correctly is also capable of distinguishing the messages (just invert \\(b'\\)).

The advantage of the adversary is modelled as a function with the security parameter \\(n\\) as input.
If this function is non-negligible (i.e. its inverse is bounded by a polynomial), then we have an attack with a non-negligible advantage:
The adversary has broken the encryption scheme.

## The Problem with Deterministic Stateless Encryption

Deterministic encryption contrary to stateful or probabilistic encryption always maps the same message to the same ciphertext for a given key.
The consequence is that it is trivial to break IND-CPA security.
We simply query the oracle for \\(E_k(LR(M_0',M_0', b))\\) at first and obtain \\(c_0\\) and then pick an arbitrary message \\(M_1'\\).
Now we can query the oracle for \\(E_k(LR(M_0',M_1',b))\\) and obtain \\(c_1\\).
If \\(c_0 = c_1 \\) then \\(b = 0\\) otherwise \\(b = 1\\).
This attack succeeds with an advantage of 1.

Concrete schemes against which this attack works are AES-ECB and textbook RSA.

Some of you might have encountered a different game definition for IND-CPA.
In this definition, the adversary is given to an oracle which merely encrypts a single message and does not have \\(b\\) as a parameter:

$$
Oracle\ E_k(M) \\
\quad return\ E_k(M)
$$

The LR oracle from the previous definition is still accessed once.
It's easy to see that our attack still works.

Students sometimes mistakenly believe that an adversary is constrained in what he can send to the encryption oracle.
These constraints exist for the decryption oracle in the CCA setting, but are absent in CPA.
The encryption oracle in CCA is also unconstrained.

## Stateful and Probabilistic Encryption

Stateful and probabilistic encryption schemes can avoid this issue since the same message can yield multiple different ciphertexts.
One example for a stateful scheme would be AES-CBC, an initialisation vector is used to initialise the state and the state is updated after every encryption.
The previously described attack would therefore not work.

An example for probabilistic encryption is padded RSA (see the RFC in the references for an example).
The message to be encrypted is prepended with a combination of random and fixed bytes, which are of course stripped after the decryption.
This too results in multiple possible ciphertexts for the same message.
However, no state is kept between different encryptions.

## References & Further Reading
Bellare, Rogaway. Introduction to Modern Cryptography: <https://web.cs.ucdavis.edu/~rogaway/classes/227/spring05/book/main.pdf>  
RFC3447 - RSAES-PKCS1-v1_5: <https://tools.ietf.org/html/rfc3447#page-23>  
