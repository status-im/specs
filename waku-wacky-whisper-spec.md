# Waku Wacky Whisper Specification

> Version: 0.1 (Draft)

> Authors: Infura "The Devil" Osaka <noreply@status.im>

Basically Infura for chat. Addresses scalability issues of Whisper by sacrificing metadata protection. Specifically for resource restricted devices like mobilephones with limited data plan.

インフラ (Infura) means infrastructure in Japanese, and 枠 (Waku) is a closely related word that means 'frame' or 'slide'.

## Rationale

Many users put a premium on bandwidth usage over metadata protection. Implementing Waku mode gives users a more performant experience, at the expense of metadata protection and requiring stronger connectivity.

For more background, please see Whisper theoretical model [report](https://htmlpreview.github.io/?https://github.com/vacp2p/research/blob/master/whisper_scalability/report.html) with associated [source code](https://github.com/vacp2p/research/tree/master/whisper_scalability).

## Roles and definitions

1. Waku-chan
2. Waku-san (aka Waku node)

A client is said to implement waku mode if it connects to a Waku node and acts as a Waku-chan. The Waku node connects to the rest of the Whisper network.

## Amendments to Whisper

Waku mode is compatible with [EIP-627](https://eips.ethereum.org/EIPS/eip-627)
and simplify extends its capabilities. It does this through a new packet code,
as well as some client specific recommendations. Two specific protocol changes
are done:

1. Modify EIP627 by adding a new packet code, e..g `Topic List [101, bytes]` where bytes is a list of N topics.

Format TBD.

2. Provide way to identify oneself as a Waku node.

Method TBD.

## Recommendations for clients

1. Avoid duplicate envelopes

To avoid duplicate envelopes, only connect to one Waku node. Benign duplicate
envelopes is an intrinsic property of Whisper which often leads to a N factor
increase in traffic, where N is the number of peers you are connected to.

2. Topic specific recommendations

TBD.
