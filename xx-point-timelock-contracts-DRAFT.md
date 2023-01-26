# Extension BOLT XX: Point Timelock Contracts

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

As can be gleamed from the description above, the PTLC counterpart to `update_add_htlc` cannot
be a single message, because we now require additional roundtrips in the protocol.

I propose we split up the PTLC initialization into three messages: `update_offer_ptlc`,
`update_accept_ptlc`, and `commitment_signed`.

### `update_offer_ptlc`

1. type: 128.1 (`update_offer_ptlc`)
2. data:
   - `channel_id`: `channel_id
   - `u64`: `id`
   - `u64`: `amount_msat`
   - `u32`: `cltv_expiry`
   - `pubkey`: PTLC commitment pubkey (e. g. `A+B+C+Z`)
   - `pubkey`: MuSig2 pubkey
   - `66*byte`: MuSig2 pubnonce
   - `1366*byte`: `onion_routing_packet`

### `update_accept_ptlc`

1. type: 128.2 (`update_accept_ptlc`)
2. data:
   - `channel_id`: `channel_id
   - `u64`: `id`
   - `pubkey`: MuSig2 pubkey
   - `66*byte`: MuSig2 pubnonce
   - `32*byte`: MuSig2 partial signature

### `commitment_signed` Extensions

1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
   1. type: 10 (`ptlc_indexed_signature`)
   2. data:
      - `u32`: `ptlc_index` (TBD whether PTLCs and HTLCs should be indexed separately or jointly)
      - `32*byte`: MuSig2 partial signature

## Sphinx Changes

### Hop Data Extensions

1. `tlv_stream`: `tlv_payload
2. types:
   1. type 10 (`ptlc_secret`)
   2. data:
      - `32*byte`: `ptlc_secret` (e. g. the `d` sent from Alice to David)

## Script Changes

The only script changes are to the `htlc_success`, now `ptlc_success` branch of the Taptree.

### Offered PTLCs

`ptlc_success`:

```
<ptlc_musig_pubkey> OP_CHECKSIG
OP_CHECKSEQUENCEVERIFY
```

#### Rationale

Because the PTLC MuSig2 signature requires buy-in from both the local and the remote parties,
the need to check individual parties' signature is obviated, so we can remove
`<remote_ptlc_pubkey> OP_CHECKSIG`.

### Accepted PTLCs

`ptlc_success`:

```
<ptlc_musig_pubkey> OP_CHECKSIG
```

#### Rationale

Because the PTLC MuSig2 signature requires buy-in from both the local and the remote parties,
the need to check individual parties' signature is obviated, so we can remove
`<local_ptlc_pubkey> OP_CHECKSIGVERIFY` and `<remote_ptlc_pubkey> OP_CHECKSIG`, and can instead
make the PTLC's check an `OP_CHECKSIG` instead of an `OP_CHECKSIGVERIFY`.