---
permalink: /spec/17
parent: Draft specs
title: 17/URL SCHEME
---

# 17/Defined Url Scheme

> Version: 0.1.0
>
> Status: Draft
>
> Authors: Aleksandar Djenic <aleksandardjenic@status.im> Michal Iskierko <michal@status.im>
>


## Table of Contents

 1. [Url Scheme](#url-scheme)
 2. [Defined url](#defined-url)

## Url scheme

- internal: "status-app://"
- external: "https://status.app"

## Defined url

| Name | Url | Description |
| ----- | ---- | ---- |
| User profile | `/u/[compressed_user_key or ens_name]` | Display user profile popup for user with `compressed_user_key` or `ens_name` |
| Community |	`/c/[compressed_community_key]` | Open community with `compressed_community_key` |
| Community channel | `/cc/[compressed_channel_key]`| Open community which has a channel with `compressed_channel_key` and makes that channel active |
| Browse | `/b/[url]` |  Open `url` in the app's browser |
| Post in channel | `/cc/[channel_key]/[message_key]` | Go to a message `message_key` in the channel `channel_key`
| Post in group | `/g/[group_key]/[message_key]` | Go to a message `message_key` in the group chat `group_key`


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
