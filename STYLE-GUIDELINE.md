---
layout: default
permalink: /style-guideline
title: STYLE-GUIDELINE
---

# Style guidelines for Status client specifications

- [Spellcheck](#spellcheck)
- [Markdown Verfication](#markdown-verification)
- [Lanaguage Mode](#language-mode)

## Spellcheck

To run the spellchecker locally, you must install [pyspelling](https://facelessuser.github.io/pyspelling/).

It can then be run with the following command:

```console
pyspelling -c spellcheck.yml
```

Words that should be ignored or are unrecognized must be added to the [wordlist](./wordlist.txt).

## Markdown Verification

We use [remark](https://remark.js.org/) to verify our markdown. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run lint
```

## Language mode

- Specifications SHOULD use formal technical language (*different from academic language*).
- Where appropriate, language SHOULD NOT use personal pronouns.
- Avoid using the [passive voice](https://en.wikipedia.org/wiki/English_passive_voice) when being specific.
- In places where the passive voice is appropriate but makes the subject ambiguous, append the passive voice with "by `subject`". Alternatively restructure the sentence to be in the active voice adding the sentence subject.

<details>
<summary>Examples:</summary>

### Personal pronouns

Informal:
>In this specification, **we** describe 

Formal:
>This specification describes 

Informal:
>If **you** want to run a Waku node and receive messages from Status clients, it must be properly configured.

Formal:
>A Waku node must be properly configured to receive messages from Status clients.

### Passive voice

Passive voice:
>a corresponding confirmation **is broadcast** by one or more peers

Active voice:
>**one or more peers broadcast** a corresponding confirmation

In the case where the object of the sentence needs to be highlighted or given prominence the passive voice is appropriate.
However, pay attention to not introduce an ambiguous subject if communicating specific information is your goal.

### Appropriate use of the passive voice

>The Batch Acknowledge packet is followed by a keccak256 hash of the envelope's batch data (raw bytes).

The subject of the sentence is "a keccak256 hash", but the sentence wants to highlight the Batch Acknowledge.

### Ambiguous subject

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