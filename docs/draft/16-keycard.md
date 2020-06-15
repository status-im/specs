---
title: 16/Keycard Usage for Wallet and Chat Keys
version: 0.1
status: Draft
authors: Roman Volosovskyi <roman@status.im>
permalink: /specs/16
redirect_from:
  - /keycard.html
---

## Table of Contents

1. [Abstract](#abstract)
2. [Definitions](#definitions)
3. [Multiaccount creation](#Multiaccount-creationrestoring)
4. [Multiaccount restoring via pairing](#Multiaccount-restoring-via-pairing)
5. [Multiaccount unlocking](#Multiaccount-unlocking)
6. [Transaction signing](#Transaction-signing)
7. [Account derivation](#Account-derivation)
8. [Reset pin](#Reset-pin)
9. [Unblock pin](#Unblock-pin)
10. [Status go calls](#Status-go-calls)
11. [Where are the keys stored?](#Where-are-the-keys-stored)
12. [Copyright](#Copyright)

## Abstract

In this specification, we describe how Status communicates with Keycard to create, store and use multiaccount.

## Definitions


| Term               | Description                                              |
| ------------------ | -------------------------------------------------------- |
| Keycard Hardwallet | [https://keycard.tech/docs/](https://keycard.tech/docs/) |
|                    |                          |


## Multiaccount creation/restoring
### Creation and restoring via mnemonic
1. `status-im.hardwallet.card/get-application-info` 
    request: `nil`
    response: `{"initialized?" false}`
2. `status-im.hardwallet.card/init-card` 
   request: `{:pin 123123}`
   response: 
   ```clojure
   {"password" "nEJXqf6VWbqeC5oN", 
    "puk" "411810112887", 
    "pin" "123123"}
   ```
3. `status-im.hardwallet.card/get-application-info` 
    request: `nil`
    response: 
    ```clojure
    {"free-pairing-slots" 5, 
     "app-version" "2.2", 
     "secure-channel-pub-key" "04e70d7af7d91b8cd23adbefdfc242c096adee6c1b5ad27a4013a8f926864c1a4f816b338238dc4a04226ab42f23672585c6dca03627885530643f1656ee69b025", 
     "key-uid" "", 
     "instance-uid" "9f149d438988a7af5e1a186f650c9328", 
     "paired?" false, 
     "has-master-key?" false, 
     "initialized?" true}
    ```
4. `status-im.hardwallet.card/pair` 
   params: `{:password "nEJXqf6VWbqeC5oN"}`
   response: `AAVefVX0kPGsxnvQV5OXRbRTLGI3k8/S27rpsq/lZrVR` (`pairing`)
   
5. `status-im.hardwallet.card/generate-and-load-keys` 
    ```clojure
    {:mnemonic "lift mansion moment version card type uncle sunny lock gather nerve math", 
     :pairing "AAVefVX0kPGsxnvQV5OXRbRTLGI3k8/S27rpsq/lZrVR", 
     :pin "123123"}
    ```
    response:
    ```clojure
    {"whisper-address" "1f29a1a60c8a12f80c397a91c6ae0323f420e609", 
     "whisper-private-key" "123123123123123", 
     "wallet-root-public-key" "04eb9d01990a106a65a6dfaa48300f72aecfeabe502d9f4f7aeaccb146dc2f16e2dec81dcec0a1a52c1df4450f441a48c210e1a73777c0161030378df22e4ae015", 
     "encryption-public-key" "045ee42f012d72be74b31a28ce320df617e0cd5b9b343fad34fcd61e2f5dfa89ab23d880473ba4e95401a191764c7f872b7af92ea0d8c39462147df6f3f05c2a11", 
     "wallet-root-address" "132dd67ff47cc1c376879c474fd2afd0f1eee6de", 
     "whisper-public-key" "0450ad84bb95f32c64f4e5027cc11d1b363a0566a0cfc475c5653e8af9964c5c9b0661129b75e6e1bc6e96ba2443238e53e7f49f2c5f2d16fcf04aca4826765d46", 
     "address" "bf93eb43fea2ce94bf3a6463c16680b56aa4a08a", 
     "wallet-address" "7eee1060d8e4722d36c99f30ff8291caa3cfc40c", 
     "key-uid" "472d8436ccedb64bcbd897bed5895ec3458b306352e1bcee377df87db32ef2c2", 
     "wallet-public-key" "0495ab02978ea1f8b059140e0be5a87aad9b64bb7d9706735c47dda6e182fd5ca41744ca37583b9a10c316b01d4321d6c85760c61301874089acab041037246294", 
     "public-key" "0465d452d12171711f32bb931f9ea26fe1b88fe2511a7909a042b914fde10a99719136365d506e2d1694fc14627f9d557da33865efc6001da3942fc1d4d2469ca1", 
     "instance-uid" "9f149d438988a7af5e1a186f650c9328"}
    ```

### Multiaccount restoring via pairing
This flow is required in case if a user want to pair a card with an existing multiaccount on it.
1. `status-im.hardwallet.card/get-application-info` 
    request: `nil`
    response: 
    ```clojure
    {"free-pairing-slots" 4, 
     "app-version" "2.2", 
     "secure-channel-pub-key" "04e70d7af7d91b8cd23adbefdfc242c096adee6c1b5ad27a4013a8f926864c1a4f816b338238dc4a04226ab42f23672585c6dca03627885530643f1656ee69b025", 
     "key-uid" "", 
     "instance-uid" "9f149d438988a7af5e1a186f650c9328", 
     "paired?" false, 
     "has-master-key?" false, 
     "initialized?" true}
    ```
2. `status-im.hardwallet.card/pair` 
   params: `{:password "nEJXqf6VWbqeC5oN"}`
   response: `AAVefVX0kPGsxnvQV5OXRbRTLGI3k8/S27rpsq/lZrVR` (`pairing`)
   
3. `status-im.hardwallet.card/generate-and-load-keys` 
    ```clojure
    {:mnemonic "lift mansion moment version card type uncle sunny lock gather nerve math", 
     :pairing "AAVefVX0kPGsxnvQV5OXRbRTLGI3k8/S27rpsq/lZrVR", 
     :pin "123123"}
    ```
    response:
    ```clojure
    {"whisper-address" "1f29a1a60c8a12f80c397a91c6ae0323f420e609", 
     "whisper-private-key" "123123123123123123123", 
     "wallet-root-public-key" "04eb9d01990a106a65a6dfaa48300f72aecfeabe502d9f4f7aeaccb146dc2f16e2dec81dcec0a1a52c1df4450f441a48c210e1a73777c0161030378df22e4ae015", 
     "encryption-public-key" "045ee42f012d72be74b31a28ce320df617e0cd5b9b343fad34fcd61e2f5dfa89ab23d880473ba4e95401a191764c7f872b7af92ea0d8c39462147df6f3f05c2a11", 
     "wallet-root-address" "132dd67ff47cc1c376879c474fd2afd0f1eee6de", 
     "whisper-public-key" "0450ad84bb95f32c64f4e5027cc11d1b363a0566a0cfc475c5653e8af9964c5c9b0661129b75e6e1bc6e96ba2443238e53e7f49f2c5f2d16fcf04aca4826765d46", 
     "address" "bf93eb43fea2ce94bf3a6463c16680b56aa4a08a", 
     "wallet-address" "7eee1060d8e4722d36c99f30ff8291caa3cfc40c", 
     "key-uid" "472d8436ccedb64bcbd897bed5895ec3458b306352e1bcee377df87db32ef2c2", 
     "wallet-public-key" "0495ab02978ea1f8b059140e0be5a87aad9b64bb7d9706735c47dda6e182fd5ca41744ca37583b9a10c316b01d4321d6c85760c61301874089acab041037246294", 
     "public-key" "0465d452d12171711f32bb931f9ea26fe1b88fe2511a7909a042b914fde10a99719136365d506e2d1694fc14627f9d557da33865efc6001da3942fc1d4d2469ca1", 
     "instance-uid" "9f149d438988a7af5e1a186f650c9328"}
   ```
   
## Multiaccount unlocking
1. `status-im.hardwallet.card/get-application-info` 
    params: 
    ```clojure
    {:pairing nil, :on-success nil}
    ```
    response: 
    ```clojure
    {"free-pairing-slots" 4, 
     "app-version" "2.2", 
     "secure-channel-pub-key" "04b079ac513d5e0ebbe9becbae1618503419f5cb59edddc7d7bb09ce0db069a8e6dec1fb40c6b8e5454f7e1fcd0bb4a0b9750256afb4e4390e169109f3ea3ba91d", 
     "key-uid" "a5424fb033f5cc66dce9cbbe464426b6feff70ca40aa952c56247aaeaf4764a9", 
     "instance-uid" "2268254e3ed7898839abe0b40e1b4200", 
     "paired?" false, 
     "has-master-key?" true, 
     "initialized?" true}
     ```
2. `status-im.hardwallet.card/get-keys`
    params:
    ```clojure
    {:pairing "ACEWbvUlordYWOE6M1Narn/AXICRltjyuKIAn4kkPXQG",
     :pin "123123"}
    ```
    response:
    ```clojure
    {"whisper-address" "ec83f7354ca112203d2ce3e0b77b47e6e33258aa", 
     "whisper-private-key" "123123123123123123123123",
     "wallet-root-public-key" "0424a93fe62a271ad230eb2957bf221b4644670589f5c0d69bd11f3371034674bf7875495816095006c2c0d5f834d628b87691a8bbe3bcc2225269020febd65a19", 
     "encryption-public-key" "0437eef85e669f800570f444e64baa2d0580e61cf60c0e9236b4108455ec1943f385043f759fcb5bd8348e32d6d6550a844cf24e57f68e9397a0f7c824a8caee2d", 
     "wallet-root-address" "6ff915f9f31f365511b1b8c1e40ce7f266caa5ce", 
     "whisper-public-key" "04b195df4336c596cca1b89555dc55dd6bb4c5c4491f352f6fdfae140a2349213423042023410f73a862aa188f6faa05c80b0344a1e39c253756cb30d8753f9f8324", 
     "address" "73509a1bb5f3b74d0dba143705cd9b4b55b8bba1", 
     "wallet-address" "2f0cc0e0859e7a05f319d902624649c7e0f48955", 
     "key-uid" "a5424fb033f5cc66dce9cbbe464426b6feff70ca40aa952c56247aaeaf4764a9", 
     "wallet-public-key" "04d6fab73772933215872c239787b2281f3b10907d099d04b88c861e713bd2b95883e0b1710a266830da29e76bbf6b87ed034ab139e36cc235a1b2a5b5ddfd4e91", 
     "public-key" "0437eef85e669f800570f444e64baa2d0580e61cf60c0e9236b4108455ec1943f385043f759fcb5bd8348e32d6d6550a844cf24e57f68e9397a0f7c824a8caee2d", 
     "instance-uid" "2268254e3ed7898839abe0b40e1b4200"}
    ```
3. `status-im.hardwallet.card/get-application-info`
    params: 
    ```clojure
    {:pairing "ACEWbvUlordYWOE6M1Narn/AXICRltjyuKIAn4kkPXQG"}
    ```
    response:
    ```clojure
    {"paired?" true, 
     "has-master-key?" true, 
     "app-version" "2.2", 
     "free-pairing-slots" 4, 
     "pin-retry-counter" 3, 
     "puk-retry-counter" 5, 
     "initialized?" true, 
     "secure-channel-pub-key" "04b079ac513d5e0ebbe9becbae1618503419f5cb59edddc7d7bb09ce0db069a8e6dec1fb40c6b8e5454f7e1fcd0bb4a0b9750256afb4e4390e169109f3ea3ba91d", 
     "key-uid" "a5424fb033f5cc66dce9cbbe464426b6feff70ca40aa952c56247aaeaf4764a9", 
     "instance-uid" "2268254e3ed7898839abe0b40e1b4200"}
    ```

## Transaction signing
1. `status-im.hardwallet.card/get-application-info`
   params: 
   ```clojure
   {:pairing "ALecvegKyOW4szknl01yYWx60GLDK5gDhxMgJECRZ+7h", 
    :on-success :hardwallet/sign}
   ```
   response:
   ```clojure
    {"paired?" true, 
     "has-master-key?" true, 
     "app-version" "2.2", 
     "free-pairing-slots" 4, 
     "pin-retry-counter" 3, 
     "puk-retry-counter" 5, 
     "initialized?" true, 
     "secure-channel-pub-key" "0476d11f2ccdad4e7779b95a1ce063d7280cb6c09afe2c0e48ca0c64ab9cf2b3c901d12029d6c266bfbe227c73a802561302b2330ac07a3270fc638ad258fced4a", 
     "key-uid" "d5c8cde8085e7a3fcf95aafbcbd7b3cfe32f61b85c2a8f662f60e76bdc100718", 
     "instance-uid" "e20e27bfee115b431e6e81b8e9dcf04c"}
    ```

2. `status-im.hardwallet.card/sign` 
   params:
   ```clojure
   {:hash "92fc7ef54c3e0c42de256b93fbf2c49dc6948ee089406e204dec943b7a0142a9", 
    :pairing "ALecvegKyOW4szknl01yYWx60GLDK5gDhxMgJECRZ+7h", 
    :pin "123123", 
    :path "m/44'/60'/0'/0/0"}
   ```
   response: `5d2ca075593cf50aa34007a0a1df7df14a369534450fce4a2ae8d023a9d9c0e216b5e5e3f64f81bee91613318d01601573fdb15c11887a3b8371e3291e894de600`

## Account derivation
1. `status-im.hardwallet.card/verify-pin` 
    params:
    ```clojure
    {:pin "123123", 
     :pairing "ALecvegKyOW4szknl01yYWx60GLDK5gDhxMgJECRZ+7h"}
    ```
    response: `3`
1. `status-im.hardwallet.card/export-key`
    params:
    ```clojure
    {:pin "123123", 
     :pairing "ALecvegKyOW4szknl01yYWx60GLDK5gDhxMgJECRZ+7h", 
     :path "m/44'/60'/0'/0/1"}
    ```
    response: `046d1bcd2310a5e0094bc515b0ec995a8cb59e23d564094443af10011b6c00bdde44d160cdd32b4b6341ddd7dc83a4f31fdf60ec2276455649ccd7a22fa4ea01d8` (account's `public-key`)

## Reset pin
1. `status-im.hardwallet.card/change-pin`
    params:
   ```clojure
   {:new-pin "111111", 
    :current-pin "222222", 
    :pairing "AA0sKxPkN+jMHXZZeI8Rgz04AaY5Fg0CzVbm9189Khob"}
   ```
   response:
   `true`
   
## Unblock pin
If user enters a wrong pin three times in a row a card gets blocked. The user can use puk code then to unblock the card and set a new pin.
1. `status-im.hardwallet.card/unblock-pin`
   params:
   ```clojure
   {:puk "120702722103", 
    :new-pin "111111", 
    :pairing "AIoQl0OtCL0/uSN7Ct1/FHRMEk/eM2Lrhn0bw7f8sgOe"}
   ```
   response
   `true`

## Status go calls
In order to use the card in the app we need to use some parts of status-go API, specifically:
1. [`SaveAccountAndLoginWithKeycard`](https://github.com/status-im/status-go/blob/b33ad8147d23a932064f241e575511d70a601dcc/mobile/status.go#L337) after multiaccount creation/restoring to store multiaccount and login into it
2. [`LoginWithKeycard`](https://github.com/status-im/status-go/blob/b33ad8147d23a932064f241e575511d70a601dcc/mobile/status.go#L373) to log into already existing account
3. [`HashTransaction`](https://github.com/status-im/status-go/blob/b33ad8147d23a932064f241e575511d70a601dcc/mobile/status.go#L492) and [`HashMessage`](https://github.com/status-im/status-go/blob/b33ad8147d23a932064f241e575511d70a601dcc/mobile/status.go#L520) for hashing transaction/message before signing
4. [`SendTransactionWithSignature`](https://github.com/status-im/status-go/blob/b33ad8147d23a932064f241e575511d70a601dcc/mobile/status.go#L471) to send transaction

## Where are the keys stored?
1. When we create a regular multiaccount all its keys are stored on device and are encrypted via key which is derived from user's password. In case if account was created using keycard all keys are stored on the card and are retrieved from it during signing into multiaccount. 
2. When we create a regular multiaccount we also create a separate database for it and this database is encrypted using key which is derived from user's password. For a keycard account we use `encryption-public-key` (returned by `status-im.hardwallet.card/get-keys`/`status-im.hardwallet.card/generate-and-load-keys`) as a password.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
