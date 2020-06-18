---
permalink: /spec/16
parent: Stable specs
title: 16/PUSH-NOTIFICATION-SERVER
---

# 16/PUSH-NOTIFICATION-SERVER

> Version: 0.1
>
> Status: Raw
>
> Authors: Andrea Maria Piana <andreap@status.im>

- [Push Notification Server](#16push-notification-server)
  - [Reason](#reason)
  - [Requirements](#requirements)
  - [Components](#components)
  - [Registering with the push notification service](#registering-with-the-push-notification-service)
  - [Re-registering with the push notification service](#re-registering-with-the-push-notification-server)
  - [Changing options](#changing-options)
  - [Unregistering from push notifications](#unregistering-from-push-notifications)
  - [Advertising a push notification server](#advertising-a-push-notification-server)
  - [Discovering a push notification server](#discovering-a-push-notification-server)
  - [Querying the push notification service](#querying-the-push-notification-server)
  - [Sending a push notification](#sending-a-push-notification)
  - [Flow](#flow)
    - [Registration process](#registration-process)
    - [Sending a notification](#sending-a-notification)
    - [Receiving a push notification](#receiving-a-push-notification)
  - [Protobuf description](#protobuf-description)
    - [PushNotificationRegister](#pushnotificationregister)
    - [PushNotificationPreferences](#pushnotificationpreferences)
    - [PushNotificationDeviceToken](#pushnotificationdevicetoken)
    - [PushNotificationOptions](#pushnotificationoptions)
    - [PushNotificationPreferencesResponse](#pushnotificationpreferencesresponse)
    - [ContactCodeAdvertisement](#contactcodeadvertisement)
    - [PushNotificationAdvertisementInfo](#pushnotificationadvertisementinfo)
    - [PushNotificationQuery](#pushnotificationquery)
    - [PushNotificationQueryInfo](#pushnotificationqueryinfo)
    - [PushNotificationQueryResponse](#pushnotificationqueryresponse)
    - [PushNotification](#pushnotification)
    - [PushNotificationRequest](#pushnotificationrequest)
    - [PushNotificationAcknowledgement](#pushnotificationacknowledgement)
  - [Partitioned topic](#partitioned-topic)
  - [Anonymous mode of operations](#anonymous-mode-of-operations)
  - [Security considerations](#security-considerations)
  - [FAQ](#faq)
  - [Changelog](#changelog)
    - [Version 0.1](#version-01)
  - [Copyright](#copyright)

## Reason

Push notifications for iOS devices and some Android devices can only be
implemented by relying on [APN service](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1) for iOS or [Firebase](https://firebase.google.com/).

This is useful for Android devices that do not support background services or that often kill the 
background service.

iOS only allows certain kind of applications to keep a connection open when in the
background, VoIP for example, which current status client does not qualify for.

Applications on iOS can also request execution time when they are in the [background](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background/updating_your_app_with_background_app_refresh)
but it has a limited set of use cases, for example it won't schedule any time
if the application was force quit, and generally is not responsive enough to implement
a push notification system.

Therefore Status provides a set of Push notification services that can be used
to achieve this functionality.

Because this can't be safely implemented in a privacy preserving manner, clients
MUST be given an option not to enable push notifications and not to send push notifications.

## Requirements

The party releasing the app needs to run their own [gorush](https://github.com/appleboy/gorush) 
publicly accessible server for sending the actual notification. This cannot be community run as it requires sharing private credentials that only status has access to.

## Components

### Gorush instance

A [gorush](https://github.com/appleboy/gorush) instance MUST be publicly 
available, this will be used only by push notification servers.

### Push notification server

A push notification server used by clients to register for push notification and
sending push notifications.

### Registering client

A status client that wants to receive push notifications

### Sending client

A status client that wants to send push notifications

## Registering with the push notification service

A client MAY register with one or more Push Notification services of their choice.

[//]: # (We can have a separate partitioned topic scheme, or individual based on PK)

A `PNR message` (Push Notification Registration) MUST be sent to the [partitioned topic](#partitioned-topic) for the
public key of the node, encrypted with this key.

The content of the message MUST contain the following [protobuf record](https://developers.google.com/protocol-buffers/):

```protobuf

message PushNotificationOptions {
  boolean enabled = 1;
  repeated string allowed_user_list = 2;
  repeated string block_user_list = 3;
  repeated string block_chat_list = 4;
}

message PushNotificationDeviceToken {
  enum TokenType {
    UNKNOWN_TOKEN_TYPE = 0;
    APN_TOKEN = 1;
    FIREBASE_TOKEN = 2;
  }
  TokenType token_type = 1;
  string token = 2;
  string installation_id = 3;
  PushNotificationPreferences preferences = 4;
}

message PushNotificationPreferences {
  repeated DeviceToken device_tokens = 1;
  uint version = 2;
  boolean unregister = 3;
}

message PushNotificationRegister {
  bytes payload = 1;
  bytes signature = 2;
}
```

A push notification server will handle the message according to the following rules:

- it MUST verify that the `signature` matches the public key of the sender.
- it MUST verify that `token_type` is supported
- it MUST verify that `token` is non empty
- it MUST verify that `installation_id` is non empty
- it MUST verify that `version` is non-zero and greater than the currently stored, if any
- it MUST verify that `device_tokens` is non empty

If `signature` does not match the public key of the sender, the message MUST be discarded.

If `token_type` is not supported, a response MUST be sent with `error` set to 
`UNSUPPORTED_TOKEN_TYPE`.

If `token`,`installation_id`,`device_tokens`,`version` are empty, a response MUST
be sent with `error` set to `MALFORMED_MESSAGE`.

If the `version` is equal or less than the currently stored version, a response MUST
be sent with `error` set to `VERSION_MISMATCH`.

If any other error occurs the `error` should be set to `INTERNAL_ERROR`.

If the request is successful, an access token MUST be generated and sent back to the client
with `success` set to `true`.

Otherwise a response MUST be sent with `success` set to `false`.

It is RECOMMENDED to use a randomly generated string for `access_token` of at least 36 characters.

`request_id` should be set to the `SHA3-256` of the signature sent by the client.

The request MUST be sent on the [partitioned topic][./10-waku-usage.md#partitioned-topic] of the sender.

The payload of the response is:

```protobuf
message PushNotificationPreferencesResponse {
  boolean success = 1;
  ErrorType error = 2;
  bytes request_id = 3;
  PushNotificationPreferences preferences = 4;
  string access_token = 5;

  enum ErrorType {
    UNKNOWN_ERROR_TYPE = 0;
    MALFORMED_MESSAGE = 1;
    VERSION_MISMATCH = 2;
    UNSUPPORTED_TOKEN_TYPE = 3;
    INTERNAL_ERROR = 4;
  }
}
```

[//]: (We can ratched and use a partitioned topic here)

A client SHOULD listen for a response sent on their [partitioned topic][./10-waku-usage.md#partitioned-topic].

If `success` is `true` the client has registered successfully.

If `success` is `false`:

- If `MALFORMED_MESSAGE` is returned, the request SHOULD not be retried without ensuring 
  that is correctly formed.
- If `INTERNAL_ERROR` is returned, the request MAY be retried, but the client MUST be 
  backoff exponentially
- If `VERSION_MISMATCH` is returned, the client SHOULD ensure that the remote version returned in `preferences` is integrated, and the version should be incremented and the request MAY be sent again.

A client MAY register with multiple Push Notification Servers in order to increase availability.

A client SHOULD make sure that all the notification services they registered with have the same information about their devices and tokens.

If no response is returned the request SHOULD be considered failed and MAY be retried with the same server or a different one, but clients MUST exponentially backoff after each trial.

If the request is successful a token is returned. This SHOULD be 
[advertised](#advertising-a-push-notification-server) as described below if no public key filtering is necessary.

## Re-registering with the push notification server

A client SHOULD re-register with the node if the APN or FIREBASE token changes.

When re-registering a client SHOULD ensure that it has the most up-to-date
`PushNotificationPreferences`, update the part relative to their `installation_id`
if necessary, increment `version` and send a `PushNotificationRegister` as described above.

Once re-registered, a client SHOULD advertise the changes.


## Changing options

This is handled in exactly the same way as re-registering above.

## Unregistering from push notifications

To unregister a client MUST send a `PushNotificationRegister` request as described
above with `unregister` in `PushNotificationPreferences` set to `true`, or removing
their device information.

The server MUST remove all data about this user if `unregistering` is `true`,
apart from the `hash` of the public key and the `version` of the last options,
in order to make sure that old messages are not processed.

A client MAY unregister from a server on explicit logout if multiple accounts 
are used on a single device.

## Advertising a push notification server

Each user registered with one or more push notification servers SHOULD
advertise periodically the push notification services that they have registered with for each device they own.

```protobuf
message PushNotificationAdvertisementInfo {
  bytes public_key = 1;
  string access_token = 2;
  string installation_id = 3;
}

message ContactCodeAdvertisement {
  repeated PushNotificationAdvertisementInfo push_notification_info = 1;
}
```

If no filtering is done based on public keys,
the access token SHOULD be included in the advertisement.
Otherwise it SHOULD be left empty.

This SHOULD be advertised on the [contact code topic](./10-waku-usage.md#contact-code-topic)
and SHOULD be coupled with normal contact-code advertisement.

Every time a user register or re-register with a push notification service, their
contact-code SHOULD be re-advertised.

Multiple servers MAY be advertised for the same `installation_id` for redundancy reasons.

## Discovering a push notification server

[//]: We could query directly the nodes on a shared topic, but for simplicity and 
bandwidth usage this is the most convenient for now.

To discover a push notification service for a given user, their [contact code topic](./10-waku-usage.md#contact-code-topic)
SHOULD be listened to.
A mailserver can be queried for the specific topic to retrieve the most up-to-date
contact code.

## Querying the push notification server

If a token is not present in the latest advertisement for a user, the server 
SHOULD be queried directly.

To query a server a message:

```protobuf
message PushNotificationQuery {
  repeated bytes public_keys = 1;
}
```

MUST be sent to the server. 

The server MUST be checking the signature of the message and comparing
that with the latest `PushNotificationPreferences` for the given public key.

If the requester has access to any access token, a response MUST be sent:

```protobuf
message PushNotificationQueryInfo {
  string access_token = 1;
  string installation_id = 2;
}

message PushNotificationQueryResponse {
  repeated PushNotificationQueryInfo info = 1;

}
```

Otherwise a response MUST NOT be sent.

Multiple queries MAY be sent for the same token from different public keys to
increase deniability.

A client MAY send request to the server using an ephemeral key at first, in order not to reveal 
their identity.

If the request is not successful, they should assume that some public key filtering is
in place at the server level and they MAY retry the request using their real identity.

## Sending a push notification

When sending a push notification, only the `installation_id` for the devices targeted 
by the message SHOULD be used.

If a message is for all the user devices, all the `installation_id` known to the client MAY be used. 

The number of devices MAY be capped in order to reduce resource consumption.

At least 3 devices SHOULD be targeted, ordered by last activity.

For any device that a token is available, or that a token is successfully queried,
a push notification message SHOULD be sent to the corresponding push notification server.

```protobuf
message PushNotification {
  string access_token = 1;
  string chat_id = 2;
}

message PushNotificationRequest {
  repeated PushNotification requests = 1;
  bytes message = 2;
  string message_id = 3;
  string ack_required = 4;
}
```

Where `message` is the encrypted payload of the message and `chat_id` is the
`SHA3-256` of the `chat_id`.
`message_id` is the id of the message (link)

If multiple server are available for a given push notification, only one notification
MUST be sent.
In such cases `ack_required` MAY bet set and the sender SHOULD be listening
on the topic derived from the first 4 bytes of `SHA3-256(ack_required)` and with
a waku AES symmetric encryption key of `ack_required`.

If no response is received 
in 3 seconds, it MAY be retried against a different server.

[//]: Can someone replay this message? a uuid could be added to avoid this

This message SHOULD be sent using an ephemeral key or unsigned.

On receiving the message, the push notification server MUST validate the access token.
If the access token is valid, a notification MUST be sent to the gorush instance with the 
following data:

```
{
  "notifications": [
    {
      "tokens": ["token_a", "token_b"],
      "platform": 1,
      "message": "You have a new message",
      "data": {
        "chat_id": chat_id,
        "message": message,
        "installation_ids": [installation_id_1, installation_id_2]
      }
    }
  ]
}
```

Where platform is `1` for IOS and `2` for Firebase, according to the [gorush
documentation](https://github.com/appleboy/gorush)

If the `ack_required` is set, a server MUST be sending back a message:

```protobuf
message PushNotificationAcknowledgement {
  string id = 1;
}
```

Where id is the `ack_required` sent by the client.
The topic and encryption key used MUST be the same as described above.

## Flow

### Registration process

- A client will generate a notification token through `APN` or `Firebase`.
- The client will [register](#registering-with-the-push-notification-service) with one or more push notification server of their choosing.
- The server should process the response and respond according to the success of the operation
- If the request is not successful it might be retried, and adjusted according to the response. A different server can be also used.
- Once the request is successful the client should [advertise](#advertising-a-push-notification-server) the new coordinates

### Sending a notification

- A client should prepare a message and extract the targeted installation-ids
- It should retrieve the most up to date information for a given user, either by
  querying a mailserver if not listening already to the given topic, or checking
  the database locally
- It should then [send](#sending-a-push-notification) a push notification according
  to the rules described
- The server should then send a request to the gorush server including all the required
  information

### Receiving a push notification

- On receiving the notification, a client can open the right account by checking the 
  `installation_id` included. The `chat_id` MAY be used to open the chat if present.
- `message` can be decrypted and presented to the user. Otherwise messages can be pulled from the mailserver if the `message_id` is no already present.

## Protobuf description

### PushNotificationRegister

A `PushNotificationRegister` is used to register with a Push Notification server.

`payload`: the protobuf encoded `PushNotificationPreferences`.

`signature`: the signature of the `payload` concatenated with the compressed `Secp256k` key of the Push Notification server. `Sig(PrivateKeyClient, payload + CompressedPublicKeyNode)`

#### Data disclosed

- Public key of the author

### PushNotificationPreferences

A push notification preferences message describes the push notification options and tokens for all the devices associated with `PublicKeyClient`.

`device_tokens`: a list of `PushNotificationPreferences`, one for each device owned by the user.
`version`: an atomically increasing number identifying the current `PushNotificationPreferences`. Any time anything is changed in the record it MUST be increased by the client, otherwise the request will not be accepted.
`unregister`: whether the account should be unregistered

#### Data disclosed

- Number of devices with push notifications enabled for a given public key
- The times a push notification record has been modified by the user

### PushNotificationDeviceToken

`PushNotificationDeviceToken` represent the token and preferences for a given device.

`token_type`: the type of token. Currently supported is `APN_TOKEN` for Apple Push 
Notification service and `FIREBASE_TOKEN` for `Firebase`.
`token`: the actual push notification token sent by `Firebase` or `APN`
`installation_id`: the [`installation_id`](./2-account.md) of the device
`preferences`: the push notification preferences for this device.

#### Data disclosed

- Type of device owned by a given user
- The `FIREBASE` or `APN` push notification token 

### PushNotificationOptions

[//]: (Any of these can be sha3 in order not to store it on the server)

`enabled`: whether the device wants to be sent push notifications
`allowed_user_list`: a list of `SHA3-256` of public keys hex encoded as a string. If non-empty
only those public keys will be issued a token.
`block_user_list`: a list of `SHA3-256` of public keys hex encoded as a string. Any public key 
in this list MUST not be issued a token.
`block_chat_list`: a list of `SHA3-256` hashes of chat ids. Any chat id in this list will not trigger a notification.

#### Data disclosed

- Hash of the public keys a user will allow/block
- Hash of the chat_id a user is not interested in for notifications

### PushNotificationPreferencesResponse

`success`: whether the registration was successful
`error`: the error type, if any
`request_id`: the `SHA3-256` hash of the `signature` of the request
`preferences`: the server stored preferences in case of an error
`access_token`: the token that needs to be used by client to send a push notification,
in case the request is successful. This is generated by the push notification server.

### ContactCodeAdvertisement

`push_notification_info`: the information for each device advertised

#### Data disclosed

- The public key of the sender

### PushNotificationAdvertisementInfo

`public_key`: the public key of the server where this device is registered
`access_token`: the access token used by the server, only if the allow/block list is non-empty
`installation_id`: the installation id of the device

#### Data disclosed

- The public key of the server the client has registered with
- Whether the user has any restriction on the public keys that are allowed to
  send notifications
- The `installation_id` of the device

### PushNotificationQuery

`public_keys`: the `SHA3-256` of the public keys the client is interested in

#### Data disclosed

- The user public key 
- The hash of the public keys the user is interested in

### PushNotificationQueryInfo

`access_token`: the access token used to send a push notification 
`installation_id`: the `installation_id` of the device associated with the `access_token`

### PushNotificationQueryResponse

`info`:  a list of `PushNotificationQueryInfo`

### PushNotification

`access_token`: the access token used to send a push notification
`chat_id`: the `SHA3-256` of the `chat_id`

### Data disclosed

- The `SHA3-256` of the `chat_id` the notification is to be sent for

### PushNotificationRequest

`requests`: a list of `PushNotification`
`message`: the encrypted message that we want to notify on
`message_id`: the [status message id](./6-payloads.md)
`ack_required`: a 32 bytes long AES key, only set if an ack is required

### Data disclosed

- The status message id for which the notification is for
- the cypher text of the message

### PushNotificationAcknowledgement

`id`: the `ack_required` string passed by the client

## Partitioned topic

This is a modification of the [partitioned topic](./10-waku-usage.md#partitioned-topic) used by clients

```golang
var partitionsNum *big.Int = big.NewInt(5000)
var partition *big.Int = big.NewInt(0).Mod(publicKey.X, partitionsNum)

partitionTopic := "push-notifications-" + strconv.FormatInt(partition.Int64(), 10)

var hash []byte = keccak256(partitionTopic)
var topicLen int = 4

if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```


## Anonymous mode of operations

An anonymous mode of operations MAY be provided by the client, where the 
responsibility of propagating information about the user is left to the client,
in order to preserve privacy.

A client in anonymous mode can register with the server using a key different 
from their identity key.
This will hide their real identity key.

This identity key is effectively a secret and SHOULD only be disclosed to clients that you the user wants to be notified by.

A client MAY advertise the access token on the contact-code topic of the key generated.
A client MAY share their identity key through [contact updates](./6-payloads.md#contact-update)

A client receiving a push notification identity key SHOULD listen to the contact code 
topic of the push notification identity key for updates.

If a client has received a push notification identity key they MAY request 
the access token using an ephemeral key when querying the server.

The method described above effectively does not share the identity of the sender
nor the receiver to the server, but MAY result in missing push notifications as
the propagation of the secret is left to the client.

This can be mitigated by [device syncing](./6-payloads.md), but not completely
addressed.

[//]: This section can be removed, for now leaving it here in order to help with the 
review process. Point can be integrated, suggestion welcome.

## Security considerations

If no anonymous mode is used, when registering with a push notification service a client discloses:

- The public key
- The devices that will receive notifications

A client MAY disclose:

- The hash of the chat_ids they want to filter out
- The hash of the public keys that the client is interested/not interested in

When running in anonymous mode, the client's public key is not disclosed.

If no anonymous mode is used, when querying a notification service a client discloses:

- The public key

In any case clients will disclose:

- The fact that is interested in sending push notification to a given client

## FAQ

### Why having ACL done at the server side and not the client?

We looked into silent notification for 
[IOS](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/pushing_background_updates_to_your_app) (android has no equivalent)
but can't be used as it's expected to receive maximum 2/3 per hour, so not our use case. There
are also issue when the user force quit the app.

### Why using an access token?

The access token is used to decouple the requesting information from the user from
actually sending the push notification.

Some ACL is necessary otherwise it would be too easy to spam users (it's still fairly 
trivial, but with this method you could allow only contacts to send you push notifications).

Therefore your identity must be revealed to the server either when sending or querying.

By using an access token we increase deniability, as the server would know 
who requested the token but not necessarily who sent a push notification. 
Correlation between the two can be trivial in some cases.

This also allows a mode of use as we had before, where the server does not propagate
info at all, and it's left to the user to propagate the token, through contact requests
for example.

### Why so many different protobuf messages?

Many are just wrappers, I have not re-used any for now for clarity.


### Why advertise with the bundle?

Advertising with the bundle allows us to piggy-back on an already implemented behavior
and save some bandwidth in cases where is not filtering by public keys 

### What's the  bandwidth impact for this?

Generally speaking, for each 1-to-1 message and group chat message you will sending
1 and `number of participants` push notifications. This can be optimized if 
multiple users are using the same push notification server. Queries have also 
a bandwidth impact but they are made only when actually needed

### What's the information disclosed?

The data disclosed with each message sent by the client is above, but for a summary:

When you register with a push notification service you may disclose:

1) Your public key
2) Which devices you have
3) The hash of the chat_ids you want to filter out
4) The has of the public keys you are interested/not interested in

When you query a notification service you may disclose:

1) Your public key
2) The fact that you are interested in sending push notification to a given user

Effectively this is fairly revealing if the user has a whitelist implemented.
Therefore sending notification should be optional.

### What prevents a user from generating a random key and getting an access token and spamming?

Nothing really, that's the same as the status app as a whole. the only mechanism that prevents
this is using a white-list as described above, but that implies disclosing your true identity to the push notification server.

### Why not 0-knowledge proofs/quantum computing

We start simple, we can iterate

### How to handle backward/forward compatibility

Most of the request have a target, so protocol negotiation can happen. We cannot negotiated
the advertisement as that's effectively a broadcast, but those info should not change and we can
always accrete the message.

### Why ack_required?

That's necessary to avoid duplicated push notifications.

Deduplication of the push notification is done on the client side, to relive a bit 
of centralization and also in order not to have to modify gorush. 

### Can I run my own node?

Sure, the methods allow that

### Can I register with multiple nodes for redundancy

Yep

### What does my node disclose?

Your node will disclose the IP address is running from, as it makes an HTTP post to 
gorush. A waku adapter could be used, but please not now.

### Does this have high-reliability requirements?

The gorush server yes, no way around it.

The rest, kind of, at least one node having your token needs to be up for you to receive notifications.
But you can register with multiple servers (desktop, status, etc) if that's a concern.

### Can someone else (i.e not status) run this?

Push notification servers can be run by anyone. Gorush can be run by anyone I take,
but we are in charge of the certificate, so they would not be able to notify status-clients.

## Changelog

### Version 0.1

Released [](https://github.com/status-im/specs/commit/)

- Initial version

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
