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

You can read about how to build this project [here](./DEVELOPMENT.md).

1. Create an issue for a new Status Improvement Proposal (SIP) or some bug that you'd like to address
2. Create a corresponding PR and ping some existing SIP editors for review

If you need help, ask in #protocol at [Status / Discord](https://discord.gg/3Exux7Y).

### Specification style guidelines

Become familiar with the [specification style guidelines](STYLE-GUIDELINE.md) to understand how you should write or amend specifications.

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

# Continuous Integration

The site is built in [Our Jenkins CI](https://ci.status.im/job/website/job/specs.status.im/) based off of `master` branch.
