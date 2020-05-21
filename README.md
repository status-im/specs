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
