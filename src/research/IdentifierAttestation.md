# Identifier attestations: web2 identifiers as token owners, off-chain tokens and other attestation use-cases ##

## Summary ##

The requirement for users to own a crypto wallet has been a longstanding user experience challenge, hindering the widespread adoption of tokens. This requirement also poses a barrier for businesses and brands aiming to issue tokens to their existing customer base. Specifically, there are two reasons why wallets are dependencies: the need for a crypto balance to pay gas costs for token transactions, and the process of token ownership.

While it's currently impossible to eliminate the need for a crypto balance for paying gas costs, we can address the second dependency. Token ownership can be redesigned to use web 2 identifiers such as email addresses and Twitter handles. This approach allows users who do not own cryptocurrency to own tokens, thereby facilitating adoption.

The key technology enabling this is identifier attestation. This refers to signed messages by reputable attestors that confirm the ownership of a web2 ID, such as an email address. A token can be issued to a commitment formed from such a web2 ID, with key information missing. This information must be provided by the token owner through a cryptographic proof. This process allows the token to be issued without the user having created a wallet or any cryptographic key. The user can later acquire such an identifier attestation and complete the proof that they own the key as well as the web2 ID. This technology was successfully deployed at DEVCON 6, an Ethereum developer conference, allowing ticket holders to mint NFT tokens long after their purchase.

This paper explores the mechanism behind this process, which involves concealing the user's identity through zero-knowledge proof. It also generalizes this approach, introducing new use-cases for attestations and establishing a standard pattern for their use on the blockchain.

## Problem definition ##

We need a mechanism to mint (issue) tokens, such as NFTs, to a user whose Ethereum address is unknown. In practice, such a user probably does not have an Ethereum address. The only way to issue the token is to link it to the user's web2 ID, such as an email address or Twitter handle, as this is the only identifier available. This is a common situation in various scenarios, such as educational programs, non-profit initiatives, online communities, government programs, and research studies, where tokens need to be issued to individuals so that they can participate in web3 activities and access web3 functionalities.

Traditionally, this is done by first making sure every user is signed up on the issuer's website, then, asking every user to create a wallet, generate an Ethereum address and link it to their web2 ID by logging in to that website and creating a signed message from the newly created Ethereum address before a deadline. Then, the brand / business creates a bloom filter to "airdrop" membership tokens to all the users who finished the above process.

As of 2023, the success rate for this traditional process is very small due to the UX burden on the user. Furthermore, the tokens are periodically, batch processed. If not, individual user must claim their token as they go, the gas cost and blockchain space abuse can be significant.

Therefore we seek a way to give tokens to the user that satisfies the following critia:

1. No wallet required: it must be possible to give tokens to the users without the users owning a wallet. In the meantime, the users who already own a wallet should still be able to access the token without significant overhead.
2. Attester-agnostic: although it will not be possible to achieve the above without the involvement of an attester, the issuer or sender of the token must not force the user to choose any specific attester. This is important because the attester will necessarily know the user's web2 ID in order to attest it, and can link the web2 ID to the user's Ethereum addreess, therefore the users should be free to choose which attester can be trusted with this confidentiality. Furthermore, we require that the user does not need to do anything - including choosing an attester - at the time a token is sent or minted to their name.
3. Privacy preserving: it must be possible to give tokens to the users by their web2 ID, without revealing that ID to the blockchain. We further requires that the user who do transactions with the said token - after they owned an Ethereum address - do not reveal their web2 ID in the process.

Furthermore, such tokens doesn't need to be issued as NFT to start with, especially if they are not traded. Examples of such use-cases are voter attestations, which act like NFT - since it is not fungible - yet only need to materialise when voting happens. These will be called off-chain tokens and will be covered later.

## Starting case: sending a Ethereum token to a user who does not have an Ethereum address ##

A user, Alice, wishes to send a token to Bob who might not have a crypto (Ethereum) address. Alice, however, knows an identifier of Bob that can be attested. e.g. Bob's email address or Bob's mobile phone number.

This section describes an approach by Alice sending a virtual and anonymous attestation, attesting to Bob's right to redeem that token from Alice through a smart contract.

## Cheque ##

We call this kind of attestation *cheque* as it is actionable with a smart contract, comparable to a traditional cheque being redeemable with a bank.

Such a *cheque* would identify Bob by an identifier and is only actionable if Bob possesses an *identifier attestation* that binds his crypto (Ethereum) address with that identifier.

## Identifier Attestation ##

The identifier - email address or mobile number† - cannot be learned from an observer with access to the Ethereum blockchain. However, it will be possible for Alice to see that Bob is redeeming cheques from other parties in the future if he redeemed the cheque he received from Alice.

We wish to ensure that only Bob (the attested owner of the identifier *and* the person holding a copy of the cheque) can redeem the cheque. Neither a malicious attestor (or someone controlling Bob's identifier) nor a man-in-the-middle who might extract the cheque, will be able to redeem the cheque alone.

Furthermore, we want to allow Bob to be able to reuse his attestation once issued to him. So that after redeeming a cheque from Alice, he can also receive a cheque from Carol and redeem it using his attestation without the need for Alice or Carol to communicate.

The protocol is secure under any composition of senders (Alices) and receivers (Bobs) based on a one-more discrete logarithm-like assumption.

# Protocol

- We assume that both Alice and Bob know Bob's identifier *i*.
- We use multiplicative group notation. Thus *G<sup>x</sup>* denotes the generator element, *G*, taken *x* times as apposed to *G·x* in some other works (when using additive group notation as normally done over elliptic curves).

## Identifier Attestation

This only needs to be done once for Bob. It can be done either before or after receiving the first cheque.

0. Bob generates an Ethereum key (if he hasn't already).

1. Bob generates a /non-signing/ privacy key *p*.

2. Bob computes a hiding to his privacy key; *S=V<sup>p</sup>*, where *V* is a generator for a large prime-order subgroup.

3. He then constructs a zero-knowledge proof that he knows the exponent *p*: He picks a random *r* and computes *R=V<sup>r</sup>*, *c=H(V, S, R)* and *d=r+c·p*. The proof is a function of *(V, S, R, d)*.

4. Bob signs a CSR (signing request) which includes the proof and his identifier *i* using his Ethereum key.

5. An attestor verifies that Bob owns the identifier, that the signature is valid and that the proof is valid by computing *c=H(V, S, R)* and verifying that *V<sup>d</sup>=R·S<sup>c</sup>*.

6. If all the checks pass then the issuer constructs a Pedersen commitment; *W=G<sup>H(i)</sup>·S* where *G* is a generator for a large prime-order subgroup.

7. Finally the attestor issues an attestation that binds Bob's Ethereum address to the commitment *W* as its subject.

## Cheque

1. Alice wishes to send Bob a certain amount of a token and knows Bob's identifier *i*. She creates a /non-signing/ one-time-key *t*, and computes a Pedersen commitment *U=G<sup>H(i)</sup>·V<sup>t</sup>*.

2. Alice writes a cheque for anyone to redeem that amount of the token from her smart contract (valid for a certain amount time period). The cheque requires a valid attestation to a commitment *W* and a zero-knowledge proof of knowledge of *x* such that *W/U=V<sup>x</sup>*.

3. Alice sends *t* and the cheque to Bob.

## Redeem the Cheque with the Attestation

Bob computes a value *x=p-t* and, in a redeeming transaction, constructs a Fiat-Shamir based Schnorr proof-of-knowledge that it knows *x* such that *W/U=V<sup>x</sup>*. That is, Bob proceeds as follows:
1. Pick random *r* and compute *R=V<sup>r</sup>*
2. Next compute *c=H(G, V, W, U, R)*
3. Finally compute *d=r+c·x*
4. Bob then signs *(G, V, W, U, R, d)* and the attestation (whose subject is *W*) and sends all these values and the signature to the smart contract.

The smart contract computes:

1. That the amount in the attestation is less than Alice's balance.
2. The attestation is a valid attestation that binds *W* to Bob (transaction sender)'s Ethereum address.
3. That the signatures is correct.
4. *c=H(G, V, W, U, R)* and verifies that *V<sup>d</sup>=R·(W/U)<sup>c</sup>*
5. That the cheque is still valid.

If all predicates are satisfied, emits the pay to Bob.

# Generalisation of the protocol

## Token attestation

This protocol has two signed messages (besides the redeeming transaction which is to be signed):

1. cheque, which is a kind of attestation to trigger an action.
2. identifier attestation, issued by an identifier attestor, which enables the beneficiary of the cheque to perform an action (redeem).

The first here does not have to be a cheque. Instead, it can be a normal attestation. Such generalisation was used in, for example, ticketing, where the first attestation attests to the rights of entering the venue corresponding to the ticket. This allows the user to use such an attestation as a blockchain token despite such a token not existing on the blockchain, so we call the first "Token Attestation" and the second "Identifier Attestation". Token Attestation's one-time-key should be more appropriately called "Token Key" since such generalisation allows it to be used for more than once. The symbol doesn't need to change since *t* can represent either one time key or token key.

## Attestations on token attestation

The "subject" of the token attestation, for example, `numero=1280`, indicating the ticket ID 1280, can be used as an identifier attestation for another 3rd party attestation, which attests something to the ticket (although the hiding might not be necessary in such a use case). Here two examples are provided:

- A speaker of the conference can invite a ticket holder to a closed-door meeting. Such an invitation is an attestation on the ticket ID. Such an invitation is, again, used as a ticket.

- CO2 coin is an issuer of carbon credits for someone who paid to offset their carbon footprint. They can issue an attestation on the ticket ID, attesting to the fact that the ticket's carbon footprint has offset. Such attestation enables access to privileged online service or discounts.

Issuing new attestations on the subject of the token attestation instead of its fingerprint (hash) has the advantage that if a specific attestation is revoked thanks to the leak of its token key, a new one - with different fingerprint - can be issued without affecting the use of such attestations.

# Implementation Issues

## Implementations based on elliptic curves

We note that despite having described the protocol using general multiplication group notation, the implementations will be based on elliptic curves. Therefore, *G, V, W, U, R* will be points on an elliptic curve where *W, U* and *R* are computed using generator points *G* and *V*. This also means that the computation in step 4 for Bob and step 4 for the smart contract will happen over the integers, modulo the curve order.

We note that the generators *G* and *V* must be constructed such that they work over the same (large) subgroup. I.e. that *V∈ &lt;G&gt;* and that **no-one** knows the discrete log of *V* to base *G* (and vice versa). Otherwise breaks become trivial.

Furthermore, it is also crucial that the curve has co-factor 1, or if not, the point *G* generates a subgroup of large prime order.

[This post](https://crypto.stackexchange.com/questions/34863/ec-schnorr-signature-multiple-standard) mentions some standards for EC-based Fiat-Shamir Schnorr proofs and thus where to look for further details.

## In the case of using a JavaScript deployed as a service

Furthermore, we note that there does not seem to be standard Javascript libraries to compute such an elliptic curve Fiat-Shamir Schnorr proof. Thus this could be allowed to be supported by a third party (specifically step 1-3 for Bob). However, if such a library is malicious, it will learn *x* and thus be able to send this value back to its author, who will, in turn, be able to impersonate Bob. This *must* not happen. Thus instead of constructing a proof of knowledge of *x* Bob uses such a library to construct a proof of knowledge of *x+w* for a random *w*. Based on this Bob will instead send *(V, S, R, d, w)* in step 3 and the server will instead verify *V<sup>d</sup>=R·S<sup>c·w</sup>*.
(A similar trick must be done for the proof used to redeem the cheque using the smart contract.)
Still, even this approach allows for a front-running displacement attack if the Javascript library sends the query to its owners who also do mining, so the miner will learn *x* and thus be able to impersonate Bob once he tried to redeem the cheque.

However, it might still be possible to easily implement this in Javascript, as SubtleCrypto.deriveKey supports the construction of an ECDH key which can be used to construct the value *r* and *t* in step 1 for Bob over an elliptic curve. Since hashing is also readily supported, step 2 can also easily be implemented. Furthermore, Javascript also supports big integer arithmetic through BigInt, which is needed to compute step 3. Thus the only real issue that might not be trivial is to extract the BigInt representation of *r* along with the curve order.
