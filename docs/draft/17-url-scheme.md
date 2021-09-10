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
> Authors: Aleksandar Djenic <aleksandardjenic@status.im>
>


## Table of Contents

 1. [Url Scheme](#url-scheme)
 2. [Defined url](#defined-url)

## Url scheme

- internal: "status-im://"
- external: "https://join.status.im"

## Defined url

| Name | Short Url | Long Url | Description |
| ----- | ---- | ---- | ---- |
| Browse | `/b/[url]` | `/browser/[url]`  | Open `url` in the app's browser |
| User profile | `/u/[user_pk or ens_name]` | `/user/[user_pk or ens_name]` | Display user profile popup for user with `user_pk` or `ens_name` |
| Private chat (1:1) | `/pm/[user_pk or ens_name]` | `/private-message/[user_pk or ens_name]` | Create or just switch to 1:1 chat with user with `user_pk` or `ens_name` |
| Public chat	| `/p/[chat_key]` | `/public/[chat_key]` | Join or switch to public chat with `chat_key` |
| Group chat | `/g/[group_name]/u/[user_pk_1 or ens_name_1]/…/u/[user_pk_19 or ens_name_19]` | `/group/[group_name]/u/[user_pk_1 or ens_name_1]/…/u/[user_pk_19 or ens_name_19]` | Open a group chat with named `group_name`, adding up to 19 participants with their `user_pk` or `ens_name`. Group chat may have up to 20 participants including the admin of a group |
| Community requests | `/cr/[community_key]` | `/community-requests/[community_key]` | Send a join community request to a community with `community_key` |
| Community |	`/c/[community_key]` | `/community/[community_key]` | Open community with `community_key` |
| Community channel | `/cc/[channel_key]` | `/community-channel/[channel_key]` | Open community which has a channel with `channel_key` and makes that channel active |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
