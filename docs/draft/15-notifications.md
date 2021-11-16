---
permalink: /draft/15
title: 15/NOTIFICATIONS
parent: Stable specs
layout: default
---

# 15/NOTIFICATIONS

> Version: 0.1
>
> Status: Draft
>
> Authors: Eric Dvorsak <eric@status.im>

## Local Notifications

A client should implement local notifications to offer notifications for any event in the app without the privacy cost and dependency on third party services. This means that the client should run a background service to continuously or periodically check for updates.

### Android

Android allows running services on the device. When the user enables notifications, the client may start a ``Foreground Service`, and display a permanent notification indicating that the service is running, as required by Android guidelines.
The service will simply keep the app from being killed by the system when it is in the background.
The client will then be able to run in the background and display local notifications on events such as receiving a message in a one to one chat.

To facilitate the implementation of local notifications, a node implementation such as `status-go` may provide a specific `notification` signal.

Notifications are a separate process in Android, and interaction with a notification generates an `Intent`. To handle intents, the `NewMessageSignalHandler` may use a `BroadcastReceiver`, in order to update the state of local notifications when the user dismisses or tap a notification. If the user taps on a notification, the `BroadcastReceiver` generates a new intent to open the app should use universal links to get the user to the right place.


### iOS

We are not able to offer local notifications on iOS because there is no concept of services in iOS. It offers background updates but they’re not consistently triggered, and cannot be relied upon. The system decides when the background updates are triggered and the heuristics aren't known.

## Why is there no Push Notifications?

Push Notifications, as offered by Apple and Google are a privacy concern, they require a centralized service that is aware of who the notification needs to be delivered to.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
