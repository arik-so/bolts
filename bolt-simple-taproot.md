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
    + [MuSig2](#musig2)
      - [Key Aggregation](#key-aggregation)
      - [Nonce Generation](#nonce-generation)
      - [Nonce Handling](#nonce-handling)
      - [Signing](#signing)
    + [Nothing Up My Sleeves Points](#nothing-up-my-sleeves-points)
  * [Design Overview](#design-overview)
  * [Specification](#specification)
    + [Feature Bits](#feature-bits)
    + [Channel Funding](#channel-funding)
      - [`accept_channel` Extensions](#-accept-channel--extensions)
      - [`accept_channel` Extensions](#-accept-channel--extensions-1)
      - [`funding_created` Extensions](#-funding-created--extensions)
      - [`funding_signed` Extensions](#-funding-signed--extensions)
      - [`funding_locked` Extensions](#-funding-locked--extensions)
    + [Cooperative Closure](#cooperative-closure)
      - [`shutdown` Extensions](#-shutdown--extensions)
      - [`closing_signed` Extensions](#-closing-signed--extensions)
    + [Channel Operation](#channel-operation)
      - [`revoke_and_ack` Extensions](#-revoke-and-ack--extensions)
      - [`channel_reestablish` Extensions](#-channel-reestablish--extensions)
    + [Funding Transactions](#funding-transactions)
    + [Commitment Transactions](#commitment-transactions)
      - [To Local Outputs](#to-local-outputs)
      - [To Remote Outputs](#to-remote-outputs)
      - [Anchor Outputs](#anchor-outputs)
    + [HTLC Scripts & Transactions](#htlc-scripts---transactions)
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

### MuSig2

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
before aggregating (`KeyAgg(KeySort(p1, p2))`)>

#### Nonce Generation

A `musig2` public nonce is a 66-byte value which is the concentration of two EC
points encoded using the compressed format:
```
nonce_1 || nonce_2
```

Each public nonce has a corresponding _secret_ nonce which is only known to the
signer.

For nonce generation, we refer to the `NonceGen` algorithm defined in the
`musig2` BIP. We only specify the set of required arguments, though a signer
can also opt to specify the additional argument in order to strengthen their
nonce.

#### Nonce Handling

Throughput this document we always assume that _fresh_ nonces are generated for
each signing attempt, and nonces are **never** written to disk. As we'll see
later, this has implications with respect to the retransmission protocol.

Note that nonces can safely be pre-generated, and even batched/pipelined ahead
of time to reduce the number of round trips in the protocol.

Finally, due to the asymmetric state of the current revocation channels, we
actually require two sets of public nonces to be exchanged by each party: one
for the local commitment, and one for the remote commitment.

#### Signing

Once public nonces have been exchanged, the `NonceAgg` algorithm is used to
combine the public nonces into the combined nonce which will be a part of the
final combined signature.

The `Sign` is used to generate a valid partial signature, with the
`PartialSigVerify` algorithm used to verify each signature.

Once all partial signatures are known, the `PartialSigAgg` algorithm is used to
combine the partial signature into a final valid BIP 340 schnorr signature.

## Design Overview

With the preliminaries out of the way, we provide a brief overview of the
Simple Taproot Channel design.

The multi-sig output becomes a single P2TR key, with `musig2` key aggregation
used to combine two keys into one. During the funding flow, public nonces are
exchanged with the first two messages.

For commitment transactions, we inherit the anchor outputs semantics, meaning
there're two outputs used for CPFP, with all scripts inheriting a `CSV 1`
restriction.

The local output of the commitment transaction uses the revocation key as the
taproot internal key, and commits to a single script path of the delay script.

The remote output of the commitment transaction uses the `simple_taproot_nums`
as the top-level internal key, and then commits to a normal remote script.

Anchor outputs use the respective local and remote keys from the commitment
transaction as the top level output, with the committed script being the `CSV
16` delay.

All HTLC scripts use the revocation key as the top-level internal key. This
allows the revocation to be executed without additional on-chain bytes, and
also reduces the amount of state nodes need to keep in order to properly handle
channel breaches.

## Specification

### Feature Bits

Inheriting the structure put forth in BOLT 9, we define a new feature bit to be
placed in the `init` message, `node_announcement` and also
`channel_announcement`. We opt to also place the bit in the
`channel_announcement` so taproot channels can be identified on the routing
level, which will eventually matter once PTLCs are fully rolled out.


| Bits  | Name                             | Description                                               | Context  | Dependencies      | Link                                  |
|-------|----------------------------------|-----------------------------------------------------------|----------|-------------------|---------------------------------------|
| 50/51 | `option_simple_taproot`| Node supports simple taproot chanenls | IC | `option_channel_type`+`option_anchors` | TODO(roasbeef): link |

The Context column decodes as follows:
 * `I`: presented in the `init` message.
 * `N`: presented in the `node_announcement` messages
 * `C`: presented in the `channel_announcement` message.
 * `C-`: presented in the `channel_announcement` message, but always odd (optional).
 * `C+`: presented in the `channel_announcement` message, but always even (required).
 * `9`: presented in [BOLT 11](11-payment-encoding.md) invoices.

The `option_simple_taproot` feature bit also becomes a defined channel type
feature bit for explicit channel negotiation.

Thought this document, we assume that `option_simple_taproot` was negotiated,
and also the `option_simple_taproot` channel type is used.

### Channel Funding

The `open_channel` and `accept_channel` messages are extended to specify a new
TLV type that houses the `musig2` public nonces:

We add `option_simple_taproot` to the set of defined channel types.

#### `open_channel` Extensions

1. `tlv_stream`: `open_channel_tlvs`
2. types:
    1. type: 2 (`local_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]
    1. type: 4 (`remote_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

##### Requirements

The sending node:

  - MUST specify the `local_musig2_pubnonce` and `remote_musig2_pubnonce` TLV
    fields.

The sending node SHOULD:

  - use the `NonceGen` algorithm defined in `bip-musig2` to generate
    `local_musig2_pubnonce` and `remote_musig2_pubnonce` to ensure it
    generates nonces in a safe manner.

The receiving node MUST fail the channel if:

  - the message doesn't include a `local_musig2_pubnonce` and
    `remote_musig2_pubnonce` value.

  - a specified public nonce cannot be parsed as two compressed secp256k1
    points

##### Rationale

For the initial Taproot channel type (`option_simple_taproot`), musig2 nonces
must be exchanged in the first round trip (open->, <-accept) to ensure that the
initiator is able to generate a valid musig2 partial signature at the next
step once the transaction to be signed is fully specified. 

The `remote_musig2_pubnonce` will be used to sign the _remote_ party's
commitment transaction, while the `local_musig2_pubnonce` will be used to sign
the local party's commitment transaction.

We require two nonces, as there're actually two messages being signed: the
local commitment and the remote commitment. If only one nonce was used, when a
party went to sign and broadcast their local commitment, they would re-use a
nonce, as the nonce was already used to sign the remote party's commitment.

#### `accept_channel` Extensions

1. `tlv_stream`: `accept_channel_tlvs`
2. types:
    1. type: 2 (`local_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]
    1. type: 4 (`remote_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

##### Requirements

The sender:

 - MUST set `local_musig2_pubnonce` and `remote_musig2_pubnonce` to the
   `musig2` public nonce as specified by the `NonceGen` algorithm of
   `bip-musig2`

 - Aggregate the received local+remote public nonces (into two combined nonces)
   using the `NonceAgg` algorithm of `bip-musig2`.

The receiver:

 - MUST reject the channel if `local_musig2_pubnonce` or
   `remote_musig2_pubnonce` are not present.

#### `funding_created` Extensions

The sender:

  - MUST derive the aggregated `musig2` public key using the exchanged `funding_pubkey`s
    - Before aggregating the two funding public keys, the keys MUST be sorted
      using the `KeySort` routine of `bip-musig2`.

  - MUST use the exchanged `local_musig2_pubnonce` and `remote_musig2_pubnonce`
    received from the remote party with the `NonceAgg` algorithm to derive the
    combined nonce

  - the `signature` field MUST be a `musig2` partial signature using the
    aggregated key defined above, and the two public nonces exchanged in prior
    messages.

  - MUST generate the attached `signature` using the combined
    `remote_musig2_pubnonce` it generated in the prior step.


The recipient:
  - MUST validate the signature as a _partial_ signature based on the
    `PartialSigVerifyInternal` (using the combined nonce for the remote party's
    commitment) algorithm of `bip-musig2`:

    - if the partial signature is invalid, MUST fail the channel

#### `funding_signed` Extensions

The sender MUST set:

  - `signature` to the valid signature, using its `funding_pubkey` for the
    initial commitment transaction, as defined in this documewnt.
    - this signature MUST be a partial signature as defined by `bip-musig2`.

The recipient:

  - MUST validate the signature as a _partial_ signature based on the
    `PartialSigVerifyInternal` algorithm of `bip-musig2`:

    - if the partial signature is invalid, MUST fail the channel

#### `funding_locked` Extensions

We add a new TLV field to the `funding_locked` message:

1. type: 36 (`funding_locked`)
2. data:
    * ...
    * [`funding_locked_tlvs`:`tlvs`]

1. `tlv_stream`: `funding_locked_tlvs`
2. types:
    1. type: 0 (`local_musig2_pubnonce`)
    2. data:
        * [`musig2_pubnonce`:`nonces`]
    1. type: 2 (`remote_musig2_pubnonce`)
    2. data:
        * [`musig2_pubnonce`:`nonces`]

Similar to the `next_per_commitment_point`, by sending the `musig2_pubnonce`
value in this message, we ensure that the remote party has our public nonce
which is required to generate a new commitment signature. Similarly, we can't
generate a new signature until we receive their `local_musig2_pubnonce` and
`remote_musig2_pubnonce`.

##### Requirements

The sender MUST:

 - MUST set `local_musig2_pubnonce` and `remote_musig2_pubnonce` to a _fresh_
   set of public nonces as specified by `bip-musig2`

The receiver MUST:

  - MUST combine their local `pubnonce` and the received `pubnonce` via the
    `NonceAgg` algorithm specified in the `musig2` BIP to received _two_
    combined nonces. This combined nonce will be used to generate the next
    commitment signature via the `Sign` algorithm specified in the `musig2`
    BIP.


### Cooperative Closure

#### `shutdown` Extensions

A new TLV field is added to the `shutdown` message:

1. type: 38 (`shutdown`)
2. data:
    * ...
    * [`shutdown_tlvs`:`tlvs`]

1. `tlv_stream`: `shutdown_tlvs`
2. types:
    1. type: 2 (`musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

Before a signature can be generated for a co-op close transaction, but sides
must exchange a fresh pair of `musig2_pubnonce` values. We package this with
the `shutdown` message so that both sides can send a `closing_signed` message
as soon as a `shutdown` message is sent and received.

Only a single nonce is needed as there's only a single message to sign: the
shared co-op close transaction.

##### Requirements

A sending node:

 - MUST set the `musig2_pubnonce` to a valid `musig2` public nonce.

A receiving node:

 - MUST verify that the `musig2_pubnonce` value is a valid `musig2` public nonce.

#### `closing_signed` Extensions

Similar to the `shutdown` message, we add a new TLV to the `closing_signed`
message's existing `tlv_stream`:

1. `tlv_stream`: `closing_signed_tlvs`
2. types:
    1. type: 2 (`musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

Both sides **MUST** provide this new TLV field.

As a fresh `musig2_pubnonce` is required to sign another fee proposal, with
each new signature, both sides send a new nonce that'll be used for the _next_
signature.

### Channel Operation

#### `commitment_signed` Extensions

A new TLV stream is added to the `commitment_signed` message:

1. type: 132 (`commitment_signed`)
2. data:
    * ...
    * [`commitment_signed_tlvs`:`tlvs`]

1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
    1. type: 2 (`remote_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

The main commitment `signature` attached in the message is now a `bip-musig2`
_partial_ schnorr signature. Using the `secnonce` generated upon connection
re-establishment or funding confirmation, the partial signature is generated
with the `Sign` algorithm specified in the `musig2` BIP.

When creating a new signature, the sender attaches their _remote_ nonce which
allows the receiver to generate an additional commitment.

##### Requirements

The sender MUST:

 - Generate the commitment signature attached using the `Sign` algorithm as
   specified in the `musig2` BIP, the aggregated public key generating during
   funding, and the local `secnonce` along with the combined aggregated public
   nonce.

 - Generate each HTLC signature according to BIP 342, as a BIP 340
   schnorr signature (non partial).

The receiver MUST:

  - Fail the channel if the received schnorr partial signature failed to verify
    using the `PartialSigVerify` paramterized with the local combined nonce
    (`local_musig2_pubnonce` of the verifying party, and the
    `remote_musig2_pubnonce` of the sending party) algorithm of `musig2`.

  - Fail the channel is _any_ of the received HTLC signatures does not validate
    according to the message extensions and sighash algorithm described in BIP
    342.

#### `revoke_and_ack` Extensions

A new TLV stream is added to the `revoke_and_ack` message:

1. type: 133 (`revoke_and_ack`)
2. data:
    * ...
    * [`revoke_and_ack_tlvs`:`tlvs`]

1. `tlv_stream`: `revoke_and_ack_tlvs`
2. types:
    1. type: 2 (`local_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

Both sides **MUST** provide this new TLV field.

Similar to sending the `next_per_commitment_point`, we also send the _next_ set
of musig2 nonces after we revoke a state. Sending these nonces allows the
remote party to propose another state transition as soon as the message is
received.

#### `channel_reestablish` Extensions

A new TLV is stream is added to the `channel_reestablish` message:

1. type: 136 (`channel_reestablish`)
2. data:
    * ...
    * [`channel_reestablish_tlvs`:`tlvs`]

1. `tlv_stream`: `channel_reestablish_tlvs`
2. types:
    1. type: 2 (`local_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]
    1. type: 4 (`remote_musig2_pubnonce`)
    2: data:
        * [`66*byte`:`nonces`]

As we want to ensure that an implementation **never** needs to write a nonce to
disk, with each connection re-establishment, we send over a set of _fresh_
nonces.

Importantly, if a `commitment_signed` message needs to be exchange, then the
latest specified nonce MUST be used. In practice, this means that when
retransmitting a state, a party must re-generate the signature using the newly
provided nonce by the remote party. By sending them anew, we also avoid having
to send the used nonces along side each signature. Remember that in order to
validate a partial signature, and even combine a partial signature, the set of
public nonces MUST be known.

If a party needs to send a `revoke_and_ack` message, the included
`local_musig2_pubnonce` will _replace_ the nonce sent with the prior
`channel_reestablish` message.

If we didn't uphold this requirement, both sides would need to write to disk an
_authenticated_ nonce (carries a digital signature) in order to convince the
remote party that they generated this nonce in the first place (remember no
nonces on disk!).

We also avoid using any sort of "deterministic" nonce scheme, as they are error
prone in the `musig2` setting, and may inadvertently lead to a signer re-using
a nonce, which means a leaked private key.


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
to achieve this property, we'll re-use the `combined_funding_key` here: its in
the best interest of the other party to enforce these semantics (mempool
pinning mitigation), so the remote party will be forced to always take the
script reveeal path.

The to remote output has the following form:

  * `OP_1 to_remote_output_key`
  * where:
    * `taproot_nums_point = 0245b18183a06ee58228f07d9716f0f121cd194e4d924b037522503a7160432f15`
    * `to_remote_output_key = taproot_nums_point + tagged_hash("TapTweak", combined_funding_key || to_remote_script_root)`
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
(output_key_y_parity | 0xc0) || taproot_nums_point
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

