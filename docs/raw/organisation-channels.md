---
permalink: /spec/17
parent: Stable specs
title: 17/ORGANISATION-CHANNELS
---

# 17/ORGANISATION-CHANNELS

> Version: 0.1
>
> Status: Raw
>
> Authors: Andrea Maria Piana <andreap@status.im>
>

- [Organisation channels](#17organisation-channels)
  - [Reason](#reason)
  - [Fetching information about an organisation](#fetching-information-about-an-organisation)
  - [Joining an organisation](#joining-an-organisation)
  - [Updating channels](#updating-channels)

## Reason

Organisation channels are a way to allow users to build communities and create curated content.

## Fetching information about an organisation

An organisation is uniquely identified by a public key.

This public key can be discovered through different means, links, indexable web pages or search directories for example.

Given that the public key of the organisation is known, information about the channels and members can be fetched by listening to a discovery topic derived by the public key:

```
   Does not matter which topic for now, as long as it does not clash with public chats.
```

The organisation SHOULD publish periodically an `OrganisationDescription` message,
at least once every 24 hours, ideally more to minimise the dependency on mailservers.


```protobuf
message ChatMessageIdentity {
  // stubbing this, it's already been prepared in a different PR
  string ens_name = 1;
  string profile_picture = 2; // not a string, more complex data structure, but you get the point
  string display_name = 3; 
  string description = 4; // this is actually unique to an organisation, and chatMessageIdentity might want to be updated, or similar

}
message OrganisationPermissions {
  enum Access {
    UNKNOWN_ACCESS = 0;
    NO_MEMBERSHIP = 1;
    INVITATION_ONLY = 2;
    ON_REQUEST = 3;
  }

  bool ens_only = 1;
  bool can_request_access = 2;
  // https://gitlab.matrix.org/matrix-org/olm/blob/master/docs/megolm.md is a candidate for the algorithm to be used in case we want to have private organisational chats, lighter than pairwise encryption using the DR, less secure, but more efficient for large number of participants
  bool private = 3;
  Access access = 4;
}

message OrganisationDescription {
  uint64 clock = 1;
  repeated bytes members = 2;
  OrganisationPermissions permissions = 3;
  ChatMessageIdentity identity = 5;
  repeated OrganisationChat chats = 6;
}

message OrganisationChat {
  string chat_id = 1;
  uint64 clock = 2;
  repeated bytes members = 3;
  OrganisationPermissions permissions = 4;
  string name = 5;
}
```

Where:

`clock` is a logical clock, described [here](link to clock value)
`members` is a list of compressed public keys that represent the members of the organisation
`identity` is a `ChatMessageIdentity` with a list of details about the organisation identity
`permissions` is a list of permissions set by the creator of the organisation
`chats` is a list of chats/(channels?) associated with the organisation


### Permissions

Permissions can be set at the organisation level or at the chat level.

Permissions at the chat level cannot override permissions at the organisation level.
(Here I will describe rules, but most of them should be simple, i.e you can't set a
channel public in an private organisation, you can have a member of a chat who is not
a member of an organisation)

`ens_only` means that only user who have an `ENS` can join the organisation
`private` indicates whether the organisation channels are encrypted with a secret
key shared only by members, otherwise the key is derivable by the chat_id
`access` indicates how membership works for this organisation. 
When is `NO_MEMBERSHIP` anyone can read and write on the channels, `private` MUST be set to false in this case.
When is `INVITE_ONLY` users MUST NOT request access, if `private` is set to `false` the channel
will be read-only. Only upon receiving an `OrganisationInvitation` a user will be able to join.
When is `ON_REQUEST` users MUST send a `OrganisationRequestJoin` to join the organisation.


## Joining an organisation

If an organisation access is set to `ON_REQUEST` a user MUST send a
message `OrganisationRequestJoin` if they want to join the organisation.
A `chat_id` MAY also be specified if they are interested in joining a particular channel
of the organisation that has access also set to `ON_REQUEST`.

An organisation MUST send back a `OrganisationRequestJoinResponse` with the `accepted`
flag set.
If set to `true` a `grant` MUST be also sent which is constructed by:

```
   Signature(public-key,org-id,clock/some other stuff)
```

```
This part I am not sure, but I think some kind of mechanism is needed, 
the correct way would be to piggy back the whole description of the org, but 
that's likely to resources intensive. So a grant is basically the next best thing, but
there might be other methods.
```

This grant MAY be attached to each message so that client can optimistically process
messages coming from this user even though the most up-to-date `OrganistationDescription` has not been fetched.

(The idea here is that we don't want to have issue with ordering of messages, for example
if I joined channel `x`, and I fetch messages from the mailserver, I might see a post from a new
  user `y` before I have fetched the org description, so I don't know whether the user 
  is actually allowed to post. By using a grant, I can optimistically process their messages,
  knowing that at some point between my latest version of the memberships and what I haven't fetched
  yet, they have been added to the org. In case they have been removed afterward, I can go back and delete those 
  messages, but in most cases I should be fine.


```
message OrganisationInvitation {
  OrganisationDescription organisation = 1;
  bytes grant = 2;
}

message OrganisationRequestJoin {
  string ens_name = 1;
  string chat_id = 2;
}

message OrganisationRequestJoinResponse {
  bool accepted = 1;
  bytes grant = 2;
}
```
