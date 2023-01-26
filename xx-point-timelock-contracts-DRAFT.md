# Extension BOLT XX: Simple Taproot Channels

## Abstract

## Motivation

## Mechanism Overview

Point Timelock Contracts differ from HTLCs in that the opening of a commitment is not the revelation
of a hash preimage, but rather the revelation of a discrete logarithm.

The mechanism is best illustrated by an example:

![](./ptlc_secrets.png)

In the above diagram, let's drill down into the communication between Carol and Dave. In an HTLC
scenario, Carol would send Dave an `update_add_htlc` message containing various fee information,
as well as an onion packet containing the information sent by Alice.

In the PTLC scenario, however, Carol must instead prepare a commitment that would be opened by
a discrete log, i. e. a signature verification tied to a specific public key.

Further, because Dave's signature must necessarily reveal to Carol how to open the commitment that
she, in turn, received from Bob, we must work with Adaptor signatures, which means that we need
one signature set up beforehand, which does not open the commitment, and one signature afterwards
that does. The delta between those signatures must be fixed beforehand.

Here's how such a scheme could look:

1. Carol sends Dave the following:
   - `A + B + C + Z`: the PTLC point that Dave will need to open
   - `P_c`: a random pubkey for a MuSig2 setup with Dave where `P_c = x_c * G`
   - `R_c`: a random MuSig2 nonce
   - Alice's onion containing `d`, which Dave can use to create his subsequent PTLC with Emily
2. Dave responds to Carol with the following:
   - `P_d`: a random pubkey for the MuSig2 setup with Carol where `P_d = x_d * G`
   - `R_d`: a random MuSig2 nonce
   - a MuSig2 partial signature: `s_d = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * x_d + r_d`
3. Finally, Carol responds to Dave:
   - complementary MuSig2 partial signature: `s_c = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * x_c`
4. Now Dave can perform that same Dance with Emily.

Carol and Dave now both have the complete signature where the following holds:

- `s' = s_c + s_d = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * (x_c + x_d) + r_c + r_d`
- `s'` is not a valid signature
- A valid signature would be `s = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * (x_c + x_d) + r_c + r_d + a+b+c+z`
- `s - s' = a + b + c + z`
- Carol cannot unilaterally produce a valid signature
- Dave can produce a valid signature if he learns `a+b+c+z`, but only by modifying `s'`
  - This guarantees that if ever a valid signature is produced, it is necessarily one from which
    Carol can infer `a+b+z`.

To "bind" this PTLC, the updated commitment transaction should require an `OP_CHECKSIGVERIFY` where
the public key `P_c + P_d` is the MuSig2 key negotiated between Carol and Dave

**Note**: For simplicity's sake, the above description does not consider that MuSig2 involves
the exchange of nonce pairs and the calculation of a linear combination.

## Messaging Changes

The PTLC counterpart to `update_add_htlc` would be `update_add_ptlc`. However, there is added
complexity stemming from the fact that a secret must be revealed by means of an arithmetic delta
between two Schnorr signatures, which means that a closed PTLC commitment necessarily also
incorporates one of the two signatures, specifically the "damaged" one, i. e. the Adaptor signature.

## Script Changes

