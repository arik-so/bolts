# Extension BOLT XX: Simple Taproot Channels

## Abstract

## Motivation

## Overview

Point Timelock Contracts differ from HTLCs in that the opening of a commitment is not the revelation
of a hash preimage, but rather the revelation of a discrete logarithm.

The mechanism is best illustrated by an example:

![](./ptlc_secrets.png)

## Messaging Changes

The PTLC counterpart to `update_add_htlc` would be `update_add_ptlc`. However, there is added
complexity stemming from the fact that a secret must be revealed by means of an arithmetic delta
between two Schnorr signatures, which means that a closed PTLC commitment necessarily also
incorporates one of the two signatures, specifically the "damaged" one, i. e. the Adaptor signature.



## Script Changes

