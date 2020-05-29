---
layout: default
permalink: /
nav_exclude: true
---

# Specifications for Status clients

![CI](https://github.com/status-im/specs/workflows/CI/badge.svg)

This repository contains a list of specifications for implementing Status and
its various capabilities.

## How to contribute

1. Create an issue for a new Status Improvement Proposal (SIP) or some bug that you'd like to address
2. Create a corresponding PR and ping some existing SIP editors for review

If you need help, ask in #protocol at Status / Discord.

### Spellcheck

To run the spellchecker locally, you must install [pyspelling](https://facelessuser.github.io/pyspelling/).

It can then be run with the following command:

```console
pyspelling -c spellcheck.yml
```

Words that should be ignored or are unrecognized must be added to the [wordlist](./wordlist.txt).

### Markdown Verification

We use [remark](https://remark.js.org/) to verify our markdown. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run lint
```

### Language mode

- Specifications SHOULD use formal technical language (*different from academic language*).
- Where appropriate, language SHOULD NOT use personal pronouns.
- Avoid using the [passive voice](https://en.wikipedia.org/wiki/English_passive_voice) when being specific.
- In places where the passive voice is appropriate but makes the subject ambiguous, prepend the passive voice with "by `subject`". Alternatively restructure the sentence to be in the active voice adding the sentence subject.

<details>
<summary>Examples:</summary>

#### Personal pronouns

Informal:
>In this specification, **we** describe 

Formal:
>This specification describes 

Informal:
>If **you** want to run a Waku node and receive messages from Status clients, it must be properly configured.

Formal:
>A Waku node must be properly configured to receive messages from Status clients.

#### Passive voice

Passive voice:
>a corresponding confirmation **is broadcast** by one or more peers

Active voice:
>**one or more peers broadcast** a corresponding confirmation

In the case where the object of the sentence needs to be highlighted or given prominence the passive voice is appropriate.
However, pay attention to not introduce an ambiguous subject if communicating specific information is your goal.

#### Appropriate use of the passive voice

>The Batch Acknowledge packet is followed by a keccak256 hash of the envelope's batch data (raw bytes).

The subject of the sentence is "a keccak256 hash", but the sentence wants to highlight the Batch Acknowledge.

#### Ambiguous subject

In many cases sentences written in passive voice may be grammatically correct but hide that the sentence lacks a specified subject.

Ambiguous:
>A message confirmation **is sent** using Batch Acknowledge 

Active specific:
>**A node sends** a message confirmation using Batch Acknowledge 

Passive specific:
>A message confirmation **is sent by a node** using Batch Acknowledge

Notice that the ambiguous sentence infers or omits the subject. Making it unclear what or who performs an action on the object of the sentence.

In the example ambiguous sentence it is not stated what or who is sending a message confirmation. 

</details>


## Spec lifecycle

Every spec has its own lifecycle that shows its maturity. We indicate this in a similar fashion to [COSS Lifecycle](https://rfc.unprotocols.org/spec:2/COSS/):

![](assets/lifecycle.png)

At present (March 30, 2020) this means stable specs are what is in v1 of the Status App. Drafts and raw are work in progress specs.

## Status Improvement Proposals (SIPs)

The main specification for writing a Status client is [1/CLIENT](https://specs.status.im/spec/1).

For all full index of all specs, see [specs.status.im](https://specs.status.im/), especially stable specs.

## Protocol Research

These are protocols that are currently being researched. These are designed to
be useful outside of Status as well. To the extent that these protocols are used
within Status clients, they will show up as SIPs in the future.

To see more on this, please visit the current home: [vac protocol](https://specs.vac.dev).
