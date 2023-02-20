---
permalink: /draft/17
title: 17/URL data
parent: Draft specs
layout: default
---

# 17/URL data

> Version: 0.1
>
> Status: Draft
>
> Authors: Felicio Mununga <felicio@status.im>

## Abstract

This document describes serialization, compression and encoding of data within URLs, so the app links could carry enough information to help users with instant previewing and verifying of their contents even prior clicking or taking and action on the content.

## Related scope

### Specs

- URL scheme

### Features

- Onboarding website
- Link preview
- Link sharing
- Deep linking
- Routing and navigation

### App entities

- Community
- Channel
- User

## Requirements

### Product

- Encode data within URL until Waku response is instant and considered reliable enough for link previewing and visiting of the onboarding website. Or until a product roadmap for alternative decentralized storage is finalized.
- A verify state while loading onboarding website until encoded data is matched against Waku response to mitigate identity attacks
- Accept only `[A-Za-z0-9_-.\u0020]` character class for textual fields
- Shortest result possible

## Implementation

## Example

- See <https://github.com/felicio/status-web/blob/34535355d46593179c997ec5ef0eda32547890aa/packages/status-js/src/utils/encode-url-data.test.ts>

### Data

#### Fields

```protobuf
syntax = "proto3";

message CommunityPreview {
 string display_name = 3;
 string description = 4;
 uint32 members_count = 5;
 string color = 8;
}

message ChannelPreview {
 string display_name = 2;
 string description = 3;
 string emoji = 5;
 string color = 6;
 CommunityPreview community = 7;
}

message UserPreview {
 string display_name = 2;
 string description = 3;
 string color = 7;
}

message URLData {
 // Community, Channel, or User
 bytes content = 1;
 // content and public key concatanated and sha256 hashed twice, returning first 4 bytes
 bytes checksum = 2;
}

```

### Encoding

- Base64url

### Compression

- Brotli

### Serialization

- Protocol buffers

## Proposals

- See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit?usp=sharing> for all
- See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit#gid=1895477181> for final two

## Discussions

- See <https://github.com/status-im/status-web/issues/327>

## Footnotes

- This specification prefers maintainability, extensibility, project familiarity and backward compatibility.
- Implementation and proposals are most effective for mid to max field lengths, not min.
- Mind that some social platforms or IMs visually trim the URLs, so they would actually not be displayed in full length. See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit#gid=1260088614>.
- Without sharing public keys with hosting servers some data like profile photos or banners cannot be included in the previews
