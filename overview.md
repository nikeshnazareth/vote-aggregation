# Vote Aggregation

## Overview

Standard Ethereum voting contracts could benefit from a mechanism to aggregate votes offline and efficiently validate and tally them on-chain. Here is a proposed design based on the Kate Polynomial Commitment scheme and BLS signatures, which are both gaining traction in the Ethereum community. I'm a few steps out of my depth, and there are parts of the design that need to be hardened, so feedback and corrections are most welcome.

## Background

### Kate Polynomial Commitments

The scheme is defined [here](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf). In short:
- A polynomial can be defined by its coefficient terms (ie. the <code>a<sub>i</sub></code> values in <code>P(x) = a<sub>0</sub> + a<sub>1</sub>·x + a<sub>2</sub>·x<sup>2</sup> + ...</code> )
    - We could also define a polynomial by its evaluations [ <code>Q(0)=a<sub>0</sub>; Q(1)=a<sub>1</sub>; Q(2)=a<sub>2</sub>; ...</code>]
    - I learned this topic from ultra Ethereum researcher Justin Drake [who claims](https://www.youtube.com/watch?v=TbNauD5wgXM) that we can work in evaluation form more easily, but I couldn't understand how the "dot product" or "bitmap" (explained below) would work, so I'm going to continue describing this scheme using coefficients
- Note that in this context, a polynomial is just a mechanism to manipulate and reason about a large number of arbitrary different values. In our case, we're interested in performing aggregate operations on votes.
- There is a mechanism to _commit_ to a polynomial. The final result is a constant-sized element. I will denote a commitment to a polynomial `P(x)` as <span style="color:green"><b>P(x)</b></span>.
- Given a commitment, you can reveal any subset of points on the polynomial, along with another proof element. Assuming supporting infrastructure (the ability to do an elliptic curve pairing, whatever that means), we can check consistency of these values in constant time.
- The commitments combine linearly:
    - eg. <code>2·<span style="color:green"><b>P(x)</b></span> + 3·<span style="color:green"><b>Q(x)</b></span> == <span style="color:green"><b>(2·P(x) + 3·Q(x))</b></span></code>
    - note: the notation I'm using is a little deceptive because the actual mathematical operations are slightly more complex, but I don't think it effects the bottleneck of the overall design.
- We can quickly validate any claimed quadratic term:
    - given <span style="color:green"><b>a</b></span>, <span style="color:green"><b>b</b></span> and <span style="color:green"><b>c</b></span> we can check whether `a·b == c`

The general strategy is to require users to pre-compute all relevant values (balances, votes, signatures, etc) and let the smart contract use these properties to verify that the equations hold and the values were calculated correctly.

There is a trusted setup that limits the maximum number of terms in the polynomial, but the linearity condition means we can mostly work around this by treating large polynomials as a psuedorandom combination of small polynomials.

### BLS Signatures

This requires users to sign messages with a BLS key pair. Signatures on the same message obey linearity, so if multiple users sign the same message, we can add all the signatures to get a combined signature <code>S<sub>sum</sub></code>. We can also add all the public keys to get a combined public key <code>PK<sub>sum</sub></code>. We only need to check the combined signature against the combined public key in order to demonstrate that all the signatures were valid.

## Polynomial Tricks

### Low-degree proofs

The trusted setup guarantees an upper limit `N` to the degree of a committed polynomial. In order to prove that a committed polynomial <span style="color:green"><b>P(x)</b></span> has a lower degree `d`, the user can calculate <code>Q(x) = x<sup>N-d</sup>·P(x)</code> and <span style="color:green"><b>Q(x)</b></span>. The smart contract can use the commitments to validate the relationship (ie. that <code>Q(x) = x<sup>N-d</sup>·P(x)</code>). This proves that the degree of `Q(x)` is limited to `N`, so the degree of `P(x)` must be limited to `d`.


### Revealing a coefficient

We have a mechanism to reveal a particular point on the polynomial, but in this design, we don't care about any of the point evaluations, we only care about the coefficients.

If we have a commitment to the polynomial <span style="color:green"><b>P(x)</b></span> and we want to reveal some coefficent <code>a<sub>i</sub></code>, we can think of the polynomial as being comprised of three parts: the lower-degree terms, the target term and the higher degree terms <code>P(x) = L(x) + a<sub>i</sub>·x<sup>i</sup> + x<sup>i+1</sup>·H(x)</code>. If the user creates a commitment to <span style="color:green"><b>L(x)</b></span> and <span style="color:green"><b>H(x)</b></span> and proves they have the right degree, the smart contract can check the equation, which validates the revealed <code>a<sub>i</sub></code>.

### Dot product

Consider two quadratic polynomials

- <code>a(x) = a<sub>0</sub> + a<sub>1</sub>x + a<sub>2</sub>x<sup>2</sup></code>
- <code>b(x) = b<sub>0</sub> + b<sub>1</sub>x + b<sub>2</sub>x<sup>2</sup></code>

We can reverse the order of coefficients in one of them, multiply the result by the other, and then investigate the second degree term:

<code>(a<sub>0</sub> + a<sub>1</sub>x + a<sub>2</sub>x<sup>2</sup>)·(b<sub>2</sub> + b<sub>1</sub>x + b<sub>0</sub>x<sup>2</sup>)<br/>
= \<lower degree terms> + (a<sub>0</sub>b<sub>0</sub> + a<sub>1</sub>b<sub>1</sub> +  a<sub>2</sub>b<sub>2</sub>)·x<sup>2</sup> + \<higher degree terms>
</code>

This term is the dot product between `a` and `b`.

More generally, given commitments to <span style="color:green"><b>a</b></span> and <span style="color:green"><b>b<sub>reversed</sup></b></span>, where both polynomials have degree `d` that is half the trusted setup size, a user can calculate and commit to <span style="color:green"><b>a·b<sub>reversed</sup></b></span>. This value can be checked by the smart contract. The user can then reveal the <code>d<sup>th</sup></code> coefficient, which will correspond to the dot product between `a` and `b`.

Obligatory XKCD:

![Obligatory XKCD](https://imgs.xkcd.com/comics/cryptography.png)

### Bitmap

Consider a commitment to a polynomial <span style="color:green"><b>P(x)</b></span> whose coefficients are claimed to be either 0 or 1. How can we test this claim?

The polynomial can be thought of as a bitmap, and `P(2)` would evaluate to the number that the bitmap represents. The Kate scheme allows the user to reveal that value directly.

For example, if <code>P(x) = 1·x<sup>2</sup> + 0·x<sup>1</sup> + 1·x<sup>0</sup></code>, then `P(2) = 5`, which is `101` in binary.

Let's define `F(x)` as the polynomial with coefficients that are all 1. Then `Q(x) = -P(x) + F(x)` should be the polynomial that inverts the coefficients of P(x) (all the 1s become 0s and vice versa).

The smart contract can use linearity to calculate <span style="color:green"><b>Q(x)</b></span>, and the user can reveal `Q(2)`. If `P(x)` is a bitmap, then `Q(2)` will be the bitwise inverse of `P(2)`.

Moreover, if `P(x)` and `Q(x)` are bitmaps, `P(1)` and `Q(1)` are the number of non-zero coefficients in each of the polynomials respectively. They should sum to `F(1)`.

**WARNING:** I have not been able to construct a non-bitmap polynomial that passes both of these tests, but that's not a proof, of course. If this scheme turns out to be viable, it will require an efficient way to test if a polynomial is a bitmap. The one I described here may be suitable. In the rest of the document I will assume such a test exists.

### Sum all coefficients

Given a commitment to a polynomial <span style="color:green"><b>P(x)</b></span>, the Kate scheme allows a user to reveal `P(1)`. This will correspond to the sum of all the coefficient terms.

## The protocol

### Global state

The protocol state is comprised of three polynomial commitments. For simplicity, I will assume the polynomial sizes are unbounded, or equivalently, the number of identities is less than half the size of the trusted setup. We should be able to handle more identities with another layer of complexity.

- <span style="color:green"><b>BLS_KEYS</b></span> is a commitment to the BLS public keys of all known users. This defines an index for each identity (eg. the fifth coefficient corresponds to identity number 5). It is initialized to zero.
- <span style="color:green"><b>SEQUENCES</b></span> is a commitment to the sequence numbers associated with the identities (to prevent replay attack). They are listed in the same order as the corresponding owner identities in `BLS_KEYS`. It is initialized to zero.
- <span style="color:green"><b>BALANCES</b></span> is a commitment to the token balances. They are listed in the same order as the corresponding owner identities in `BLS_KEYS`. It is initialized to zero.

### Registering

To add a public key to <span style="color:green"><b>BLS_KEYS</b></span>:

- the user picks a zero coefficient on the polynomial (say position `i`) and constructs a proof that reveals it (to be zero)
- the user sends this proof and a new public key `PK` to the contract
- the contract validates the proof
- the contract constructs a commitment to <span style="color:green"><b>x<sup>i</sup>·PK</b></span>
- the contract updates <code><span style="color:green"><b>BLS_KEYS</b></span> = <span style="color:green"><b>BLS_KEYS</b></span> + <span style="color:green"><b>x<sup>i</sup>·PK</b></span></code>

### Mint

To mint tokens for the identity at position `i`, using the identity at position `j`:

- the user constructs proofs to reveal
    - the current balance at position `i` of <span style="color:green"><b>BALANCES</b></span>
    - the public key at position `j` of <span style="color:green"><b>BLS_KEYS</b></span>
    - the sequence number at position `j` of <span style="color:green"><b>SEQUENCES</b></span>
- the user constructs a message consisting of these proof and the number of new tokens `t`
- the user signs the message and sends it to the contract
- the contract validates the proofs
- the contract validates that identity `j` can mint tokens to position `i`
- the contract checks to see if the balance would overflow
- the contract constructs a commitment to <span style="color:green"><b>x<sup>i</sup>·t</b></span>
- the contract updates <code><span style="color:green"><b>BALANCES</b></span> = <span style="color:green"><b>BALANCES</b></span> + <span style="color:green"><b>x<sup>i</sup>·t</b></span></code>
- the contract retrieves the commitment to <span style="color:green"><b>x<sup>i</sup></b></span>
- the contract updates <code><span style="color:green"><b>SEQUENCES</b></span> = <span style="color:green"><b>SEQUENCES</b></span> + <span style="color:green"><b>x<sup>i</sup></b></span></code>

### Burn

The mechanics of burning tokens is the same as minting them except the token commitment is subtracted from <span style="color:green"><b>BALANCES</b></span> instead of added.

### Transfer

A token transfer can be thought of as a burn and mint operation.

### Vote Initialization

To initialize a vote:

- the contract saves a copy of <span style="color:green"><b>BALANCES</b></span>. This effectively creates a snapshot that can be used for voting, while the original commitment can still be updated to allow token transfers.
- the contract creates a commitment <span style="color:green"><b>HAS_VOTED</b></span> and initializes it to zero.
- the contract creates a `tally` variable and initializes it to zero.

### Public Voting (Option 1)

I will assume each vote can be represented by -1 or 1 and that the final result is the sum of these votes, weighted by the balances. Note that using linearity, it's easy to map a simple set like this into a different set (eg. we can take 0s and 1s and map them to -1 and 1s). The scheme can also be extended with more options.

To conduct a vote:

- Every user `i` publishes a message containing their vote (eg. "On question abcd, I vote +1"), along with their signatures <code>S<sub>i</sub></code>
- an aggregator takes an arbitrary subset of the ballots and divides them by the particular vote (the -1s are together, the +1s are together)
- the aggregator creates <code>SELECTION<sub>+1</sub></code> and <code>SELECTION<sub>-1</sub></code> polynomials that "select" the corresponding identities.
    - these are both bitmap polynomials, where a coefficient of 1 implies the identity matches the polynomial
    - the coefficients are specified in reverse order, so they can be used in a dot product (explained above)
- the aggregator creates commitments to <span style="color:green"><b>SELECTION<sub>+1</sub></b></span> and <span style="color:green"><b>SELECTION<sub>-1</sub></b></span>
- the aggregator creates proofs that they are bitmaps
- the aggregator creates a commitment to a <span style="color:green"><b>SIGNATURES</b></span> polynomial that contains all the signatures in the same order as <span style="color:green"><b>BLS_KEYS</b></span> (unused signatures can be set to zero)
- note that the dot product between `SIGNATURES` and <code>SELECTION<sub>+1</sub></code> will be the sum of the signatures of the +1 voters. They all voted on the same message so BLS signature aggregation would apply. Similarly, the dot product between `BLS_KEYS` and <code>SELECTION<sub>+1</sub></code> will be the sum of their public keys. The same analysis applies to the -1 voters using the <code>SELECTION<sub>-1</sub></code> polynomial. The aggregator calculates all these values, as well as the commitments required to prove that they're accurate (<span style="color:green"><b>SIGNATURES·SELECTION<sub>+1</sub></b></span>, <span style="color:green"><b> BLS_KEYS·SELECTION<sub>+1</sub></b></span>, <span style="color:green"><b>SIGNATURES·SELECTION<sub>-1</sub></b></span>, <span style="color:green"><b>BLS_KEYS·SELECTION<sub>-1</sub></b></span>)
- similarly, the dot product between `BALANCES` and (<code>SELECTION<sub>+1</sub> - SELECTION<sub>-1</sub></code>) is the signed sum of the votes, weighted by the corresponding balances. In other words, it's the tally represented by this subset of votes. The aggregator calculates this value and the proof element <span style="color:green"><b>BALANCES·(SELECTION<sub>+1</sub> - SELECTION<sub>-1</sub>)</b></span>
- the aggregator sends all these values, commitments and proofs to the contract
- the contract verifies all the proofs
- the contract updates <code><span style="color:green"><b>HAS_VOTED</b></span> = <span style="color:green"><b>HAS_VOTED</b></span> + <span style="color:green"><b>SELECTION<sub>+1</sub></b></span> + <span style="color:green"><b>SELECTION<sub>-1</sub></b></span></code>
    - note that the selection polynomials have reversed coefficients so the identities are marked off in reverse order. That's okay because we're just looking for duplicates.
- the contract verifies that <span style="color:green"><b>HAS_VOTED</b></span> is still a bitmap (otherwise, a voter will be double-counted)
- the contract verifies both of the aggregated BLS signatures
- the contract verifies that the expected message was signed by both groups
- the contract scales the selection polynomial commitments by their values and combines them:
  - <code><span style="color:green"><b>APPROVE</b></span> = <span style="color:green"><b>SELECTION<sub>+1</sub></b></span></code>
  - <code><span style="color:green"><b>REJECT</b></span> = -<span style="color:green"><b>SELECTION<sub>-1</sub></b></span></code>
  - <code><span style="color:green"><b>VOTE</b></span> = <span style="color:green"><b>APPROVE</b></span> + <span style="color:green"><b>REJECT</b></span> </code>
- the contract confirms this is consistent with the tally proof
- the contract updates the `tally` variable accordingly

The advantage of this scheme is that all the relevant calculations are performed off-chain by the aggregator. The contract can validate the effect a large number of ballots with a small number of constant-sized proof elements and constant-time checks. It should also be noted that this scheme is still censorship resistant because each update only requires a subset of the votes. Any votes that are ignored by the aggregator can be submitted in their own update.

### Hidden Voting (Option 2)

The previous option only works if we're willing to aggregate votes in the open, which means voters can be influenced by the ballots of other voters. Many existing systems use a commit-reveal process to temporarily hide the votes. To that end, it's worth noting that the original [Kate paper](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf) includes a mechanism for masking a commitment and then revealing it. Our additional constraint is that the different terms in the eventual `VOTE` polynomial correspond to different voters, so they need to be masked and unmasked individually without sacrificing the ability to perform aggregate calculations.

#### Pedersen construction

If the trusted setup supports it, it's possible to include a random number along with a Kate commitment in order to hide the commitment. I'll denote a blinded commitment to `P(x)` with a blinding factor `r` as <b><span style="color:green">P(x)</span><span style="color:purple">|r</span></b>. For our purposes, there are a few important properties of this blinded commitment:

- it is the same size as the original commitment, so blinding has minimal impact on bandwidth and computational requirements
- a user can reveal `r`, which allows anyone else to recover <span style="color:green"><b>P(x)</b></span>
- it is binding. The user can't later "reveal" a different random value in order to retroactively change the commitment.
- mathematically, it is equivalent to <span style="color:green"><b>P(x) + λr</b></span>, where λ is unknown (it's an intrinsic property of the trusted setup)

Additionally, if I understand the mathematics correctly (which should be validated by a mathematician):

- instead of revealing `r`, the user could reveal an unblinding commitment <b><span style="color:green">0</span><span style="color:purple">|-r</span></b>. This could be added to the original commitment to recover <span style="color:green"><b>P(x)</b></span>. This allows for aggregate unblinding operations.
- if the user revealed a different blinding factor `x` with the same format (<b><span style="color:green">0</span><span style="color:purple">|x</span></b>), they cannot control (except by trial and error) the "recovered" <span style="color:green"><b>P(x)</b></span>. So if <span style="color:green"><b>P(x)</b></span> has enough structure that random modifications would be detectable, a malicious user could not change their commitment.
- however, if a malicious user revealed an arbitrary unblinding commitment (with no enforced structure) that was naively added to their original commitment, they could control the "recovered" value, thereby changing their commitment.
- therefore, the contract needs to validate the structure of unblinding commitments.

#### Procedure

To conduct a vote:

- the user with identity `i` in `BLS_KEYS` hashes a message containing their vote (eg. "On question abcd, I vote +1") and uses this as the coefficient in a single-term polynomial with degree `i` (ie. <code>P<sub>i</sub>(x) = H(m)·x<sup>i</sup></code>)
- they publish the blinded commitment <b><span style="color:green">P<sub>i</sub>(x)</span><span style="color:purple">|r<sub>i</sub></span></b>
- they also publish a low-degree and high-degree proof that the commitment only affects one term
- any subset `α` of users  can coordinate offline to sum their commitments to create <b><span style="color:green">VOTES<sub>α</sub></span><span style="color:purple">|r<sub>α</sub></span></b> and sign the result
    - this will be a polynomial where the coefficient of the <code>i<sup>th</sup></code> term is user `i`'s blinded vote
    - since blinded votes will all be unique, users cannot aggregate their signatures if they only sign their component. This is why they sign the whole polynomial, which requires more coordination. If some users are non-cooperative, they can be excluded from the subset.
    - users will need to check the degree proofs to ensure other users are not interfering with their own term in the polynomial
- as before (see Option 1 for details)
    - an aggregator creates a `SELECTION` polynomial that "selects" the corresponding identities, and a `SIGNATURES` polynomial with all the signatures in the relevant positions
    - they provide all the necessary commitments and proofs so that the contract can verify (in aggregate) all relevant voter signatures and that <span style="color:green"><b>HAS_VOTED</b></span> is updated accordingly
- Ideally, as each subset `γ` of votes is submitted, the contract can add them to a global <b><span style="color:green">VOTES</span><span style="color:purple">|r</span></b> commitment
    - however, we need to ensure <b><span style="color:green">VOTES<sub>γ</sub></span><span style="color:purple">|r<sub>γ</sub></span></b> doesn't affect any other voters
    - this seems like it should be possible, but my current design has a flaw:
        - the contract constructs <b><span style="color:green">INVERTED_SELECTION<sub>γ</sub></span></b>, which flips the bits of <b><span style="color:green">SELECTION<sub>γ</sub></span></b> (see the _Bitmap_ section above)
        - the users proves (and the contract validates) that the dot product <b><span style="color:green">INVERTED_SELECTION<sub>γ</sub>·VOTES<sub>γ</sub></span><span style="color:purple">|r<sub>γ</sub></span></b> is zero
        - this does not prove these values are zero, because they may be non-zero values that add to zero
    - if this cannot be resolved, we could require voter reveals to occur within their initial commitment subset
        - this would add some storage overhead because we would need to save the individual <b><span style="color:green">SELECTION<sub>γ</sub></span></b> and <b><span style="color:green">VOTES<sub>γ</sub></span><span style="color:purple">|r<sub>γ</sub></span></b> commitments, and handle them separately
        - fortunately, it may not increase the number of elliptic curve pairing and BLS signature operations because an aggregate reveal that spans multiple <b><span style="color:green">SELECTION<sub>γ</sub></span></b> polynomials would involve checking multiple polynomial equations, and these can be pseudorandomly combined into a single equation:
            - ie. for any two equations `a·b == c` and `x·y == z`, the contract could generate a pseudorandom `r` and then combine them into `a·b + r(x·y) == c + rz`
- In the _REVEAL_ phase, any subset of voters can coordinate offline to sign and reveal their unblinding commitment
    - the aggregator should supply selection polynomials <code>SELECTION<sub>+1</sub></code> and <code>SELECTION<sub>-1</sub></code> that select the users within each voting category. The overall `SELECTION` is the sum of these two values.
    - we can use the same techniques to validate the signatures, prevent double-counting (with a <span style="color:green"><b>HAS_REVEALED</b></span> bitmap) and ensure users don't interfere with other votes within the same subset
        - as before, we may need to save each revealed subset to prevent users affect votes outside their subset
        - however, in this phase, it may be unnecessary because tampered votes are recoverable:
            - tampering with votes that have already been revealed has no effect, since the vote has already been counted
            - assuming the consistency checks hold, tampering with a masked vote commitment `i` simply adds a public value to the existing random blinding factor <code>r<sub>i</sub></code>. This value could be incorporated into the subsequent unblinding commitment.
    - as noted above, the contract needs to confirm that the unblinding commitment has the form <b><span style="color:green">0</span><span style="color:purple">|r(x)</span></b>
        - the user just needs to supply  <b><span style="color:green">r(x)</span></b> and prove that <code><b><span style="color:green">0</span><span style="color:purple">|r(x)</span></b> == <b><span style="color:green">0</span><span style="color:purple">|1</span></b>·<b><span style="color:green">r(x)</span></b></code> 
        - note the both sides of this equation are equivalent to <b><span style="color:green">λ·r(x)</span></b>
    - the contract can unblind the subset of votes by adding the unblinding commitment to <b><span style="color:green">VOTES</span><span style="color:purple">|r</span></b>
    - the aggregator can reveal and prove the sum of <code>SELECTION<sub>+1</sub></code>, which is the number of voters `p` in this subset who voted _Approve_. Similarly, they can reveal and prove the number of voters `n` who voted _Reject_.
    - the aggregator should also provide proofs that
        - <code>SELECTION<sub>+1</sub>·VOTES == p·H("On question abcd, I vote +1")</code>
        - <code>SELECTION<sub>-1</sub>·VOTES == p·H("On question abcd, I vote -1")</code>
    - as before, the contract can update `tally` with <code>BALANCES.(SELECTION<sub>+1</sub> - SELECTION<sub>-1</sub>)</code>
