# Extension BOLT XX: Simple Taproot Channels

Authors:
  * Olaoluwa Osuntokun <roasbeef@lightning.engineering>
  * Eugene Siegel <eugene@lightning.engineering>

Created: 2022-04-20

# Table of Contents

- [Introduction](#introduction)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Preliminaries](#preliminaries)
    + [Pay-To-Taproot-Outputs](#pay-to-taproot-outputs)
      - [Tapscript Tree Semantics](#tapscript-tree-semantics)
      - [BIP 86](#bip-86)
      - [Taproot Key Path Spends](#taproot-key-path-spends)
      - [Taproot Script Path Spends](#taproot-script-path-spends)
    + [MuSig2 & Associated Changes](#musig2--associated-changes)
      - [Key Aggregation](#key-aggregation)
      - [Nonce Generation](#nonce-generation)
      - [Nonce Handling](#nonce-handling)
      - [Signing](#signing)
    + [Nothing Up My Sleeves Points](#nothing-up-my-sleeves-points)
  * [Design Overview](#design-overview)
  * [Specification](#specification)
    + [Feature Bits](#feature-bits)
    + [Channel Funding](#channel-funding)
      - [`accept_channel` Extensions](#accept_channel-extensions)
      - [`funding_created` Extensions](#funding_created-extensions)
      - [`funding_signed` Extensions](#funding_signed-extensions)
      - [`channel_ready`/`channel_reestablish` Extensions](#channel_readychannel_reestablish-extensions)
    + [Cooperative Closure](#cooperative-closure)
      - [`closing_signed` Extensions](#closing_signed-extensions)
    + [Channel Operation](#channel-operation)
      - [`commitment_signed` Extensions](#commitment_signed-extensions)
      - [`revoke_and_ack` Extensions](#revoke_and_ack-extensions)
    + [Funding Transactions](#funding-transactions)
    + [Commitment Transactions](#commitment-transactions)
      - [To Local Outputs](#to-local-outputs)
      - [To Remote Outputs](#to-remote-outputs)
      - [Anchor Outputs](#anchor-outputs)
    + [HTLC Scripts & Transactions](#htlc-scripts--transactions)
      - [Offered HTLCs](#offered-htlcs)
      - [Accepted HTLCs](#accepted-htlcs)
      - [HTLC Second Level Transactions](#htlc-second-level-transactions)
        * [HTLC-Success Transactions](#htlc-success-transactions)
        * [HTLC-Timeout Transactions](#htlc-timeout-transactions)
- [Appendix](#appendix)
- [Footnotes](#footnotes)
- [Test Vectors](#test-vectors)
- [Acknowledgements](#acknowledgements)

# Introduction

## Abstract


The activation of the Taproot soft-fork suite enables a number of updates to
the Lightning Network, allowing developers to improve the privacy, security,
and flexibility of the system. This document specifies extensions to BOLTs 2,
3, and 5 which describe a new Taproot based channels to update to the Lightning
Network to take advantage of _some_ of these new capabilities. Namely, we
mechanically translate the current funding and commitment design to utilize
`musig2` and the new tapscript tree capabilities.

## Motivation

The activation of Taproot grants protocol developers with a number of new tools
including: schnorr, musig2, scriptless scripts, PTLCs, merkalized script trees
and more. While it's technically possible to craft a _single_ update to the
Lightning Network to take advantage of _all_ these new capabilities, we instead
propose a step-wise update process, with each step layering on top of the prior
with new capabilities. While the ultimately realization of a more advanced
protocol may be delayed as a result of this approach, packing up smaller
incremental updates will be easier to implement and review, and may also
compress the timeline to an initial taproot based channel roll out.

In this document, we start by revising the most fundamental component of a
Lightning Channel: the funding output. By first porting the funding output to
Segwit V1 (P2TR) `musig2` based output, we're able to re-anchor all channels in
the network with the help of dynamic commitments. Once those channels are
re-anchored, dynamic commitments allows developers to ship incremental changes
to the commitment transaction, HTLC structure, and overall channel structure,
all without additional on-chain transactions.

By restricting the surface area of the _initial_ update, we aim to expedite the
initial roll-out while also providing a solid base for future updates without
blocking off any interesting design paths. Implementing and shipping a
constrained update also gives implementers and opportunity to get more familiar
with the new tooling and capabilities, before tackling more advanced protocol
designs.

## Preliminaries

In this section we lay out some preliminary concepts, protocol flows, and
notation that'll be used later in the core channel specification.

### Pay-To-Taproot-Outputs

The Taproot soft fork package (BIPs [340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki), [341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki), and [342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)) introduced a new [Segwit](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)
witness version: v1. This new witness version utilize a new standardized public
key script template:
```
OP_1 <taproot_output_key>
```

The `taproot_output_key` is a 32-byte x-only `secp256k1` public key, with all
digital signatures based off of the schnorr signature scheme described in BIP
340.

A `taproot_output_key` commits to an internal key, and optional script root via
the following mapping:
```
taproot_output_key = taproot_internal_key + tagged_hash("TapTweak", taproot_internal_key || script_root)*G
```

It's important to note that while `taproot_output_key` is serialized as a
32-byte public key, in order to properly spend the script path, the parity
(even or odd) of the public key must be remembered.

The `taproot_internal_key` is also a BIP 340 public key, ideally derived anew
for each output. The `tagged_hash` scheme is described in BIP 340, we briefly
define the function as:
```
tagged_hash(tag, msg) = SHA256(SHA256(tag) || SHA256(tag) || msg)
```

#### Tapscript Tree Semantics

The `script_root` is the root of a Tapscript tree as defined in BIP 341. A tree
is composed of `tap_leaves` and `tap_branches`.

A `tap_leaf` is a two tuple of (`leaf_version`, `leaf_script`). A `tap_leaf` is serialized as:
```
leaf_version || compact_size_of(leaf_script) || leaf_script
```
where `compact_size_of` is the variable length integer used in the Bitcoin p2p
protocol.

The digest of a `tap_leaf` is computed as:
```
tagged_hash("TapLeaf", leaf_encoding)
```

A `tap_branch` can commit to either a `tap_leaf` or `tap_branch`. Before
hashing, a `tap_branch` sorts the two `tap_node` arguments based on
lexicographical ordering:
```
tagged_hash("TapBranch", sort(node_1, node_2))
```

A tapscript tree is constructed by hashing each pair of leaves into a
`tap_branch`, then combining these branches with each other until only a single
hash remains. There're no strict requirements at the consensus level for tree
construction, allowing systems to "weight" the tree by placing items that occur
more often at the top of the tree. A simple algorithm that attempts to [produce
a balanced script tree can be found here](https://github.com/btcsuite/btcd/blob/99e4e00345017a70eadc4e1d06353c56b23bb15c/txscript/taproot.go#L618-L776).

#### BIP 86

A `taproot_output_key` doesn't necessarily _need_ to commit to a `script_root`,
unless the `script_root` is being revealed, a normal key (skipping the tweaking
operation) can be used. In many cases it's also nice to prove to a 3rd party
that the output can only be spent with a top-level signature and not also a
script path.

[BIP 86](https://github.com/bitcoin/bips/blob/master/bip-0086.mediawiki) defines a taproot output key derivation scheme that wherein only the internal
key is committed to without a script root:
```
taproot_output_key = taproot_internal_key + tagged_hash("TapTweak", taproot_internal_key)*G
```

We use BIP 86 whenever we want to ensure that a script path spend isn't
possible, and an output can only be spent via a key path.

We recommend that in any instances where a normal unencumbered output is used
(cooperative closures, etc) BIP 86 is used as well.

#### Taproot Key Path Spends

A taproot key path spend allows one to spend a taproot output without revealing
the script root, which may or may not exist.

The witness of a keypath spend is simply a 64 or 65 byte schnorr signature. A
signature is 64 bytes if the new `SIGHASH_DEFAULT` alias for `SIGHASH_ALL` is
being used. Otherwise a single byte for the sighash type is used as normal:
```
<key_path_sig>
```

If an output commits to a script root (or uses BIP 86), a private key will need
to be tweaked in order to generate a valid signature, and negated if the
resulting output key has an odd y coordinate (see BIP 341 for details).

#### Taproot Script Path Spends

A script path spend is executed when a tapscript leaf is revealed. A script
path spends has three components: a control block, the script leaf being
revealed, and the valid witness.

A control block has the following serialization:
```
(output_key_y_parity | leaf_version) || internal_key || inclusion_proof
```

The first segment uses a spare bit of the `leaf_version` to encode the parity
of the output key, which is checked during control block verification. The
`internal_key` is just that. The `inclusion_proof` is the series of sibling
hashes in the path from the revealed leaf all the way up to the root of the
tree. In practice if one hashes the revealed leaf, and each 32-byte hash of the
inclusion proof together, they'll reach the script root if the proof was valid.

If only a single leaf is committed to in a tree, then the `inclusion_proof`
will be absent.

The final witness to spend a script path output is:
```
<witness1> ... <witnessN> <leaf_script> <control_block>
```

### MuSig2 & Associated Changes

Musig2 is a multi-signature scheme based on schnorr signatures. It allows N
parties to construct an aggregated public key, and then generate a single
multi-signature that can only be generated with all the parties of the
aggregated key. In practice we use this to allow the funding output of
channel to look just like any other P2TR output.

A musig2 signing flow has 4 stages:

  1. First each party generates a public nonce, and exchanges it with every
    other party.

  2. Next public keys are exchanged by each party, and combined into a single
    aggregated public key.

  3. Next, each party generates a _partial_ signature and exchanges it with
    every other party.

  4. Finally, once all partial signatures are known, they can be combined into a
    valid schnorr signture, which can be veirfied using BIP 340.

Steps 1&2 may be performed out of order, or concurrently. In our case only two
parties exist, so as soon as one party knows both partial signatures, they can
be combined into a final signature.

#### Key Aggregation

We refer to the of `KeyAgg` algorithm [bip-musig2](https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki)
for key aggregation. In order to avoid sending extra ordering information, we
always assume that both keys are sorted first with the `KeySort` algorithm
before aggregating (`KeyAgg(KeySort(p1, p2))`).

#### Nonce Generation

A `musig2` secret nonce is the concatenation of two random, 32-byte integers.

A `musig2` public nonce is technically the concatenation of two public keys, each representing the EC-point
corresponding to its secret integer, thus resulting in a 66-byte value:

```
point_1 || point_2
```

For nonce generation, we refer to the `NonceGen` algorithm defined in the
`musig2` BIP. We only specify the set of required arguments, though a signer can also opt to specify the additional
argument in order to strengthen their nonce.

#### Nonce Handling

Due to the asymmetric state of the current revocation channels, two sets of public nonces need to be exchanged by each
party: one for the local commitment, and one for the remote commitment.

The nonce for the local commitment MUST be sent a priori, thus enabling the counterparty to create a valid partial
signature.

The nonce for the remote commitment _could_ be sent a priori, too, but for the purposes of simplicity and
disambiguation, MUST be sent alongside the partial signature. To that end, we will create a new data type that will be
reused in multiple TLV extensions described further down in this document.

#### Signing

Once public nonces have been exchanged, the `NonceAgg` algorithm is used to combine the public nonces into the aggregate
nonce which will be a part of the final signature.

The `Sign` method is used to generate a valid partial signature, with the
`PartialSigVerify` algorithm used to verify each signature.

Once all partial signatures are known, the `PartialSigAgg` algorithm is used to aggregate the partial signature into a
final, valid
[BIP 340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) Schnorr signature.

##### Partial Signature Encoding

A standard `musig2` partial signature is simply the encoding of the `s` value, which is a 32-byte-element modulo the
order of the group.

Throughout this specification, we will be using a custom type, `PartialSignatureWithNonce`, comprised of the
aforementioned `s` value, along with the 66-byte-encoding of the public nonce used to sign remote commitments:

```
s || point_1 || point_2
```

## Design Overview

With the preliminaries out of the way, we provide a brief overview of the
Simple Taproot Channel design.

The multisig output becomes a single P2TR key, with `musig2` key aggregation used to combine two keys into one.

For commitment transactions, we inherit the anchor output semantics, meaning there are two outputs used for CPFP, with
all other scripts inheriting a `1 CSV`
restriction.

The local output of the commitment transaction uses the revocation key as the
taproot internal key, and commits to a single script path of the delay script.

The remote output of the commitment transaction uses the `combined_funding_key`
as the top-level internal key, and then commits to a normal remote script.

Anchor outputs use the respective two parties' funding keys as the top-level internal key committing to the
script `16 CSV`. If a local or remote output is missing from the commitment, its corresponding anchor must not be
present, either.

All HTLC scripts use the revocation key as the top-level internal key. This allows the revocation to be executed without
additional on-chain bytes, and also reduces the amount of state that nodes need to keep in order to properly handle
channel breaches.

## Specification

### Feature Bits

Inheriting the structure put forth in BOLT 9, we define a new feature bit to be
placed in the `init` message, `node_announcement` and also
`channel_announcement`. We opt to also place the bit in the
`channel_announcement` so taproot channels can be identified on the routing
level, which will eventually matter once PTLCs are fully rolled out.


| Bits  | Name             | Description                           | Context | Dependencies                             | Link                 |
|-------|------------------|---------------------------------------|---------|------------------------------------------|----------------------|
| 30/31 | `option_taproot` | Node supports simple taproot chanenls | IN      | `option_channel_type` + `option_anchors` | TODO(roasbeef): link |

The Context column decodes as follows:
 * `I`: presented in the `init` message.
 * `N`: presented in the `node_announcement` messages
 * `C`: presented in the `channel_announcement` message.
 * `C-`: presented in the `channel_announcement` message, but always odd (optional).
 * `C+`: presented in the `channel_announcement` message, but always even (required).
 * `9`: presented in [BOLT 11](11-payment-encoding.md) invoices.

The `option_taproot` feature bit also becomes a defined channel type feature bit for explicit channel
negotiation.

Throughout this document, we assume that `option_taproot` was negotiated, and also the `option_taproot`
channel type is used.

### Channel Funding

`n_a_L`: Alice's local secret nonce

`n_B_R`: Bob's remote public nonce

![](taproot_channel_open.jpg)

We add `option_taproot` to the set of defined channel types.

#### `accept_channel` Extensions

1. `tlv_stream`: `accept_channel_tlvs`
2. types:
   1. type: 4 (`next_local_nonce`)
   2. data:
      * [`66*byte`:`public_nonce`]

##### Requirements

The sender:

- MUST set `next_local_nonce` to the `musig2` public nonce used to sign local commitments as specified by the `NonceGen`
  algorithm of `bip-musig2`.

The recipient:

- MUST reject the channel if `next_local_nonce` is absent.
- MUST store `next_local_nonce` until they send the `funding_created` message.
  - SHOULD only store `next_local_nonce` in RAM, and not persist it to disk.

#### `funding_created` Extensions

1. `tlv_stream`: `funding_created_tlvs`
2. types:
   1. type: 2 (`partial_signature_with_nonce`)
   2. data:
      * [`98*byte`: `partial_signature || public_nonce`]
   3. type: 4 (`next_local_nonce`)
   4. data:
      * [`66*byte`: `public_nonce`]

##### Requirements

The sender:

- MUST sort the exchanged `funding_pubkey`s using the `KeySort` algorithm from `bip-musig2`.
- MUST compute the aggregated `musig2` public key from the sorted `funding_pubkey`s using the `KeyAgg` algorithm
  from `bip-musig2`.
- MUST generate a unique nonce to combine with the `next_local_nonce` previously received in
  `accept_channel` using `NonceAgg` from `bip-musig2`.
- MUST use the generated secret nonce and the calculated aggregate nonce to construct a
  `musig2` partial signature for the sender's remote commitment using the `Sign` algorithm from `bip-musig2`.
- MUST include the partial signature and the public counterpart of the generated nonce in
  the `partial_signature_with_nonce` field.
- MUST generate a second unique nonce to send in the `next_local_nonce` field. This nonce MUST be used to sign the sender's
  local commitment using the `Sign` algorithm from `bip-musig2`.

The recipient:

- MUST fail the channel if `partial_signature_with_nonce` is absent.
- MUST fail the channel if `next_local_nonce` is absent.
- MUST compute the aggregate nonce from:
  - the `next_local_nonce` field the recipient previously sent in `accept_channel`
  - the `public_nonce` included as part of the `partial_signature_with_nonce` field
- MUST verify the `partial_signature_with_nonce` field using the `PartialSigVerifyInternal`
  algorithm of `bip-musig2`:
  - if the partial signature is invalid, MUST fail the channel
- MUST store `next_local_nonce` until they send the `funding_signed` message.
  - SHOULD only store `next_local_nonce` in RAM, and not persist it to disk.

#### `funding_signed` Extensions

1. `tlv_stream`: `funding_signed_tlvs`
2. types:
   1. type: 2 (`partial_signature_with_nonce`)
   2. data:
      * [`98*byte`: `partial_signature || public_nonce`]

##### Requirements

The sender:

- MUST sort the exchanged `funding_pubkey`s using the `KeySort` algorithm from `bip-musig2`.
- MUST compute the aggregated `musig2` public key from the sorted `funding_pubkey`s using the `KeyAgg` algorithm
  from `bip-musig2`.
- MUST generate a unique nonce to combine with the `next_local_nonce` previously received in
  `funding_created` using `NonceAgg` from `bip-musig2`.
- MUST use the generated secret nonce and the calculated aggregate nonce to construct a
  `musig2` partial signature for the sender's remote commitment using the `Sign` algorithm from `bip-musig2`.
- MUST include the partial signature and the public counterpart of the generated nonce in
  the `partial_signature_with_nonce` field.

The recipient:

- MUST fail the channel if `partial_signature_with_nonce` is absent.
- MUST compute the aggregate nonce from:
  - the `next_local_nonce` field the recipient previously sent in `funding_created`
  - the `public_nonce` included as part of the `partial_signature_with_nonce` field
- MUST verify the `partial_signature_with_nonce` field using the `PartialSigVerifyInternal`
  algorithm of `bip-musig2`:
  - if the partial signature is invalid, MUST fail the channel

#### `channel_ready`/`channel_reestablish` Extensions

We add a new TLV field to the `channel_ready` and `channel_reestablish` messages:

1. `tlv_stream`: `channel_ready_tlvs`/`channel_reestablish_tlvs`
2. types:
   1. type: 4 (`next_local_nonce`)
   2. data:
      * [`66*byte`: `public_nonce`]

Similar to the `next_per_commitment_point`, by sending the `next_local_nonce` value in this message,
we ensure that the remote party has our public nonce which is required to generate a new
commitment signature.

##### Requirements

The sender:

- MUST set `next_local_nonce` to a fresh, unique `musig2` nonce as specified by `bip-musig2`

The recipient:

- MUST fail the channel if `next_local_nonce` is absent.
- MUST store `next_local_nonce` until they send the next `commitment_signed` message.
  - SHOULD only store `next_local_nonce` in RAM, and not persist it to disk.

### Cooperative Closure

#### `closing_signed` Extensions

We add a new TLV to the `closing_signed`
message's existing `tlv_stream`:

1. `tlv_stream`: `closing_signed_tlvs`
2. types:
   1. type: 2 (`partial_signature_with_nonce`)
   2. data:
      * [`98*byte`: `partial_signature || public_nonce`]
   3. type: 4 (`next_local_nonce`)
   4. data:
      * [`66*byte`: `public_nonce`]

Both sides **MUST** provide this new TLV field.

As a fresh `next_local_nonce` is required to sign another fee proposal, with each new signature, both sides send a new
nonce that'll be used for the _next_ signature.

Once all partial signatures are known, the `PartialSigAgg` algorithm is used to aggregate the partial signature into a
final, valid
[BIP 340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) Schnorr signature.

#### Requirements

The sender:

- MUST set the original, non-TLV `signature` field to a 0-byte-array of length 64.
- MUST retrieve the `musig2` public key previously aggregated from the sorted `funding_pubkey`s using the `KeyAgg` algorithm
  from `bip-musig2`.
- MUST generate a unique nonce to combine with the `next_local_nonce` previously received in the latest
  of `channel_ready`, `channel_reestablish`, `revoke_and_ack`, or `closing_signed` using `NonceAgg` from `bip-musig2`.
- MUST use the generated secret nonce and the calculated aggregate nonce to construct a
  `musig2` partial signature for the sender's remote commitment using the `Sign` algorithm from `bip-musig2`.
- MUST include the partial signature and the public counterpart of the generated nonce in
  the `partial_signature_with_nonce` field.
- MUST generate a second unique nonce to send in the `next_local_nonce` field. This nonce MUST be used to sign the sender's
  local commitment using the `Sign` algorithm from `bip-musig2`.

The recipient:

- MUST fail the channel if `signature` is non-zero.
- MUST fail the channel if `partial_signature_with_nonce` is absent.
- MUST fail the channel if `next_local_nonce` is absent.
- MUST compute the aggregate nonce from:
  - the `next_local_nonce` field the recipient previously sent in the latest of `channel_ready`,
    `channel_reestablish`, `revoke_and_ack`, or `closing_signed`
  - the `public_nonce` included as part of the `partial_signature_with_nonce` field
- MUST verify the `partial_signature_with_nonce` field using the `PartialSigVerifyInternal`
  algorithm of `bip-musig2`:
  - if the partial signature is invalid, MUST fail the channel
- MUST store `next_local_nonce` until they, in turn, send _their_ `closing_signed` message.
  - SHOULD only store `next_local_nonce` in RAM, and not persist it to disk.

### Channel Operation

`n_a_L`: Alice's local secret nonce

`n_B_R`: Bob's remote public nonce

![](taproot_channel_operation.jpg)

#### `commitment_signed` Extensions

A new TLV stream is added to the `commitment_signed` message:

1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
   1. type: 2 (`partial_signature_with_nonce`)
   2. data:
      * [`98*byte`: `partial_signature || public_nonce`]

##### Requirements

The sender:

- MUST set the original, non-TLV `signature` field to a 0-byte-array of length 64.
- MUST retrieve the `musig2` public key previously aggregated from the sorted `funding_pubkey`s using the `KeyAgg` algorithm
  from `bip-musig2`.
- MUST generate a unique nonce to combine with the `next_local_nonce` previously received in the latest
  of `channel_ready`, `channel_reestablish`, or `revoke_and_ack` using `NonceAgg` from `bip-musig2`.
- MUST use the generated secret nonce and the calculated aggregate nonce to construct a
  `musig2` partial signature for the sender's remote commitment using the `Sign` algorithm from `bip-musig2`.
- MUST include the partial signature and the public counterpart of the generated nonce in
  the `partial_signature_with_nonce` field.
- MUST compute each HTLC signature according to
  [BIP 342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki), as a BIP 340 Schnorr
  signature (non-partial).

The recipient:

- MUST fail the channel if `signature` is non-zero.
- MUST fail the channel if `partial_signature_with_nonce` is absent.
- MUST compute the aggregate nonce from:
  - the `next_local_nonce` field the recipient previously sent in the latest of `channel_ready`,
    `channel_reestablish`, or `revoke_and_ack`
  - the `public_nonce` included as part of the `partial_signature_with_nonce` field
- MUST verify the `partial_signature_with_nonce` field using the `PartialSigVerifyInternal`
  algorithm of `bip-musig2`:
  - if the partial signature is invalid, MUST fail the channel
- MUST fail the channel if _any_ of the received HTLC signatures is invalid according to BIP 342.

#### `revoke_and_ack` Extensions

A new TLV stream is added to the `revoke_and_ack` message:

1. `tlv_stream`: `revoke_and_ack_tlvs`
2. types:
   1. type: 4 (`next_local_nonce`)
   2. data:
      * [`66*byte`: `public_nonce`]

Similar to sending the `next_per_commitment_point`, we also send the _next_ `musig2` nonce after we revoke a
state. Sending this nonce allows the remote party to propose another state transition as soon as the message is
received.

##### Requirements

The sender:

- MUST generate a unique nonce to send in the `next_local_nonce` field. This nonce MUST be used to sign the sender's
  local commitment using the `Sign` algorithm from `bip-musig2`.

The recipient:

- MUST fail the channel if `next_local_nonce` is absent.
- MUST store `next_local_nonce` until they send the next `commitment_signed` message.
  - SHOULD only store `next_local_nonce` in RAM, and not persist it to disk.
- If the local nonce generation is non-deterministic and the recipient co-signs commitments only upon
  pending broadcast:
  - MUST **securely** store the local nonce.

### Funding Transactions

For our Simple Taproot Channels, `musig2` is used to generate a single
aggregated public key.

The resulting funding output script takes the form:
`OP_1 funding_key`

where:

  * `funding_key = combined_funding_key + tagged_hash("TapTweak", combined_funding_key)*G`
  * `combined_funding_key = musig2.KeyAgg(musig2.KeySort(pubkey1, pubkey2))`

The funding key is derived via the recommendation in BIP 341 that states: "If
the spending conditions do not require a script path, the output key should
commit to an unspendable script path instead of having no script path. This can
be achieved by computing the output key point as `Q = P +
int(hashTapTweak(bytes(P)))G`". We refer to BIP 86 for the specifics of this
derivation.

By using the musig2 sorting routine, we inherit the lexicographical key
ordering from the base segwit channels. Throughout the lifetime of a channel
any keys aggregated via musig2 also are assumed to use the `KeySort` routine

### Commitment Transactions

#### To Local Outputs

For the simple taproot commitment type, we use the same flow of revocation or
delay, but instead left the revocation case into the taproot key-spend case.

The new output has the following form:

  * `OP_1 to_local_output_key`
  * where:
    * `to_local_output_key = revocationpubkey + tagged_hash("TapTweak", revocationpubkey || to_delay_script_root)`
    * `to_delay_script_root = tapscript_root([to_delay_script])`
    * `to_delay_script` is the delay script:
        ```
        <local_delayedpubkey> OP_CHECKSIG
        <to_self_delay> OP_CHECKSEQUENCEVERIFY OP_DROP
        ```

The parity (even or odd) of the y-coordinate of the derived
`to_local_output_key` MUST be known in order to derive a valid control block.

The `tapscript_root` routine constructs a valid taproot commitment according to
BIP 341+342. Namely, a leaf version of `0xc0` MUST be used. As there's only a
single script, one can derive the tapscript root as:
```
tapscript_root = tagged_hash("TapLeaf", 0xc0 || compact_size_of(to_delay_script) || to_delay_script)
```

In the case of a commitment breach, the `to_delay_script_root` can be used
along side `<revocationpubkey>` to derive the private key needed to sweep the
top-level key spend path. A valid witness is then just:
```
<revocation_sig>
```

In the case of a routine force close, the script path must be revealed so the
broadcaster can sweep their funds after a delay. The control block to spend is
only `33` bytes, as it just includes the internal key (along with the y-parity
bit and leaf version):
```
delay_control_block = (output_key_y_parity | 0xc0) || revocationpubkey
```

A valid witness is then:
```
<local_delayedsig> <to_delay_script> <delay_control_block>
```

As with base channels, the `nSequence` field must be set to `to_self_delay`.

#### To Remote Outputs

As we inherit the anchor output semantics we want to ensure that the remote
party can unilaterally sweep their funds after the 1 block CSV delay. In order
to achieve this property, we'll re-use the `_funding_key` here: its in
the best interest of the other party to enforce these semantics (mempool
pinning mitigation), so the remote party will be forced to always take the
script reveeal path.

The to remote output has the following form:

  * `OP_1 to_remote_output_key`
  * where:
    * `to_remote_output_key = combined_funding_key + tagged_hash("TapTweak", combined_funding_key || to_remote_script_root)`
    * `to_remote_script_root = tapscript_root([to_remote_script])`
    * `to_remote_script` is the remote script:
        ```
        <remotepubkey> OP_CHECKSIG
        OP_CHECKSEQUENCEVERIFY
        ```

This output can be swept by the remote party with the following witness:
```
<remote_sig> <to_remote_script> <to_remote_control_block>
```

where `to_remote_control_block` is:
```
(output_key_y_parity | 0xc0) || combined_funding_key
```

#### Anchor Outputs

For simple taproot channels (`option_simple_taproot`) we'll maintain the same
form as base segwit channels, but instead will utilize `local_delayedpubkey`
and the `remotepubkey` rather than the multi-sig keys as that's no longer
revealed due to musig2.

An anchor output has the following form:

  * `OP_1 anchor_output_key`
  * where:
    * `anchor_internal_key = remotepubkey/local_delayedpubkey`
    * `anchor_output_key = anchor_internal_key + tagged_hash("TapTweak", anchor_internal_key || anchor_script_root)`
    * `anchor_script_root = tapscript_root([anchor_script])`
    * `anchor_script`:
        ```
        OP_16 OP_CHECKSEQUENCEVERIFY
        ```

This output can be swept by anyone after 16 blocks with the following witness:
```
<anchor_script> <anchor_control_block>
```

where `anchor_control_block` is:
```
(output_key_y_parity | 0xc0) || anchor_internal_key
```

`nSequence` needs to be set to 16.

### HTLC Scripts & Transactions

#### Offered HTLCs

We retain the same second-level HTLC mappings as base channels. The main
modifications are using the revocation public key as the internal key, and
splitting the timeout and success paths into individual script leaves.

An offered HTLC has the following form:

  * `OP_1 offered_htlc_key`
  * where:
    * `offered_htlc_key = revocation_pubkey + tagged_hash("TapTweak", revocation_pubkey || htlc_script_root)`
    * `htlc_script_root = tapscript_root([htlc_timeout, htlc_success])`
    * `htlc_timeout`:
        ```
        <local_htlcpubkey> OP_CHECKSIGVERIFY
        <remote_htlcpubkey> OP_CHECKSIG
        ```
    * `htlc_success`:
        ```
        OP_SIZE 32 OP_EQUALVERIFY OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        <remote_htlcpubkey> OP_CHECKSIG
        OP_CHECKSEQUENCEVERIFY
        ```

In order to spend a offered HTLC, via either script path, an `inclusion_proof`
must be specified along with the control block. This `inclusion_proof` is
simply the `tap_leaf` hash of the path _not_ taken.

#### Accepted HTLCs

Accepted HTLCs inherit a similar format:

  * `OP_1 accepted_htlc_key`
  * where:
    * `accepted_htlc_key = revocation_pubkey + tagged_hash("TapTweak", revocation_pubkey || htlc_script_root)`
    * `htlc_script_root = tapscript_root([htlc_timeout, htlc_success])`
    * `htlc_timeout`:
        ```
        <remote_htlcpubkey> OP_CHECKSIG
        OP_CHECKSEQUENCEVERIFY
        <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
        ```
    * `htlc_success`:
        ```
        OP_SIZE 32 OP_EQUALVERIFY OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        <local_htlcpubkey> OP_CHECKSIGVERIFY
        <remote_htlcpubkey> OP_CHECKSIG
        ```

Note that there is no `OP_CHECKSEQUENCEVERIFY` in the offered HTLC's timeout path
and in the accepted HTLC's success path. This is not needed here as these paths
require a remote signature that commits to the sequence, which needs to be set at 1.

In order to spend an accepted HTLC, via either script path, an
`inclusion_proof` must be specified along with the control block. This
`inclusion_proof` is simply the `tap_leaf` hash of the path _not_ taken.

TODO(roasbeef): specify full witnesses?

#### HTLC Second Level Transactions

For the second level transactions, we retain the existing structure: a 2-of-2
multi-spend going to a CTV delayed output. Like the normal HTLC transactions,
we also place the revocation key as the internal key, allowing easy breach
sweeps with each party only retaining the specific script root.

An HTLC-Success or HTLC-Timeout is a one-in-one-out transaction signed using
the `SIGHASH_SINGLE|SIGHASH_ANYONECANPAY` flag. These transactions always have
_zero_ fees attached, forcing them to be aggregated with each other and a
change input.

##### HTLC-Success Transactions

A HTLC-Success transaction has the following structure:

  * version: 2
  * locktime: 0
  * input:
    * txid: commitment_tx
    * vout: htlc_index
    * sequence: 1
    * witness: `<remotehtlcsig> <localhtlcsig> <preimage> <htlc_success_script> <control_block>`
  * output:
    * value: htlc_value
    * script:
      * OP_1 htlc_success_key
      * where:
        * `htlc_success_key = revocation_pubkey + tagged_hash("TapTweak", revocation_pubkey || htlc_script_root)`
        * `htlc_script_root = tapscript_root([htlc_success])`
        * `htlc_success`:
        ```
        <local_delayedpubkey> OP_CHECKSIG
        <to_self_delay> OP_CHECKSEQUENCEVERIFY OP_DROP
        ```

##### HTLC-Timeout Transactions

A HTLC-Timeout transaction has the following structure:

  * version: 2
  * locktime: cltv_expiry
  * input:
    * txid: commitment_tx
    * vout: htlc_index
    * sequence: 1
    * witness: `<remotehtlcsig> <localhtlcsig> <htlc_timeout_script> <control_block>`
  * output:
    * value: htlc_value
    * script:
      * OP_1 htlc_timeout_key
      * where:
        * `htlc_timeout_key = revocation_pubkey + tagged_hash("TapTweak", revocation_pubkey || htlc_script_root)`
        * `htlc_script_root = tapscript_root([htlc_timeout])`
        * `htlc_timeout`:
        ```
        <local_delayedpubkey> OP_CHECKSIG
        <to_self_delay> OP_CHECKSEQUENCEVERIFY OP_DROP
        ```

# Appendix

# Footnotes

# Test Vectors

# Acknowledgements

The commitment and HTLC scripts are heavily based off of t-bast's [Lightning Is
Getting Taprooty Scriptless-Scripty](https://github.com/t-bast/lightning-docs/blob/master/taproot-updates.md#lightning-is-getting-taprooty-scriptless-scripty) document.

AJ Towns proposed a more ambitious ["one shot" upgrade which includes PTLCs](https://lists.linuxfoundation.org/pipermail/lightning-dev/2021-October/003278.html)
and a modified revocation scheme which this proposal takes inspiration from.

