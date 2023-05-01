
# A Simple Web of Trust for Authenticated Relay Operator IDs (AROIs)

This is a work-in-progress draft, significant portions might change in a non-compatible way.

![trust-aroi](https://user-images.githubusercontent.com/11165848/235495697-1424d733-5971-4e4a-b73e-b8f0808d8b7e.png)


## Motivation

Tor users are facing persistent malicious actors
repeatedly running large fractions of the tor network's capacity to exploit them 
[[1]](https://nusenu.medium.com/the-growing-problem-of-malicious-relays-on-the-tor-network-2f14198af548)
[[2]](https://nusenu.medium.com/how-malicious-tor-relays-are-exploiting-users-in-2020-part-i-1097575c0cac)
[[3]](https://nusenu.medium.com/tracking-one-year-of-malicious-tor-exit-relay-activities-part-ii-85c80875c5df)
[[4]](https://nusenu.medium.com/is-kax17-performing-de-anonymization-attacks-against-tor-users-42e566defce8).
Detecting all malicious tor network capacity is not practically feasible using active scanners
in many cases since attackers have moved from attacking all connections to more targeted approaches where only
users of specific domains (that are not necessarily known to defenders) are exploited and some malicious relay operators
also run non-exit relays.
Therefore we propose to publish relay operator trust information to subsequently have the possibility to limit the fraction
and impact of unknown tor network capacity by consuming trust information.

Trust in the context of this document is a proxy for a link between community members. Trust information is unidirectional.
For example a local hackerspace might trust its community members to run tor relays without malicious intent.

additional context:
* [Proposal for Relay Operator Trust Relationship Information](https://gitlab.torproject.org/tpo/community/relays/-/issues/68)
* rC3-2021: [Towards a more Trustworthy Tor Network](https://media.ccc.de/v/rc3-2021-chaosstudiohamburg-475-towards-a-more-trustworthy-tor-network) (with en subtitles and German translation)

## Scope of this Document

This document describes how trust information is published, can be discovered, retrieved and validated.
It does not introduce any new requirements for tor relay operators. 

## Goal

The high level goal is to limit the impact of malicious tor relays on users
and to increase the trustworthiness of the tor network for tor users while also increasing the cost for malicious actors, especially the cost for recovery of malicious relay capacity after getting detected. To achieve that goal we need
information about whether a operator is known/trusted or completely unknown (no links to known entities).

On a more technical level the goal is to establish a simple protocol to publish and consume trust information about authenticated relay operator IDs (AROI)
and to make discovery and enumeration of trust relationships possible.
The authenticated relay operator ID (AROI) is the proven operator domain [[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md). 
As of April 2023 over 65% [[6]](https://nusenu.github.io/OrNetStats/#top-10-exit-operators-with-a-proven-domain) 
of the tor exit capacity has adopted the AROI and it is the only available specified operator identifier
that can not be arbitrarily spoofed and can be verified automatically.

Trust is assigned to relay operator IDs (AROIs) and not individual relay fingerprints for scalability reasons and
so new or changed relays for a known operator get the same level of trust
without requiring any update to published trust information.

The trust scheme supports any relay type (guards, middle and exit relays) but is not for bridges.

## Design goals

- simplicity: trust information is published by setting a torrc option and it gets published via a relay's signed descriptor
- low entry barrier for publishing trusted AROIs
- scalability: The design should support at least 10 000 AROIs
- trust relationships are public by design
- trust information must be machine readable
- bridges are out of scope and not supported

## Threat Model

### Adversary Goals

Adversaries want to maximize their network fraction of the overall tor network capacity
and run their attacks without getting detected. If they get detected they aim at having no increased setup cost
when compared to the previous attempt.

### Adversary Capabilities

* Adversaries can add tor relays at significant scale and proof 
[[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
their relationship to a domain under their control.
* They can NOT modify another relay operator's torrc options.
* They can NOT manipulate another relay operator's proof file
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md#well-knowntor-relayrsa-fingerprinttxt)
by other operators.
* They can NOT create valid DNSSEC-signed DNS records at a domain not under their control.

### Trust Information Consumer Goals

Trust information consumer - tor directory authorities could be an example consumer - want to learn about trusted AROIs, discover trust paths between AROIs 
and detect manipulated trust information.

## Roles

### Tor Directory Authorities (DA)

DAs publish trusted AROIs in the tor consensus.
By publishing an AROI a DA asserts that they trust the operator - identified by their AROI - to run tor relays without malicious intent. 

Trust is binary. There is no notion of "some" trust.
Consumers of trust information can use the consensus to find trust paths between relay operators (identified by their AROI).

### Relay Operators

Relay operators are identified by their AROI
[[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md).

Relay operators can also publish trust information and trust information consumers can make use of it to find relationships between relay operators. 
Trust information published by relay operators has the same meaning (asserts that the published operators are running relays without malicious intent)
but since relays do not publish a consensus, a relay publishes trust information via its descriptor.
An operator can publish trust information via a relay's torrc configuration file option.

## Publishing Trusted AROIs

Relay operators optionally publish trusted relay operator IDs (AROIs) via a torrc option
to allow trust information consumers (for example DAs) to discover trust paths (relationships) between
relay operators.

Relay operators can specify trusted AROIs in their torrs file. An AROI is optionally followed by a ":" and the recursion flag "r".
This recursion flag tells the consumer whether the given AROI is also trusted to publish AROIs itself.

Publishers MUST only specify AROIs where they are confident that the operator does operate relays **without** malicious intent.

Example torrc section :
```
# example with recursion flag enabled:
TrustedAROI example.com:r
# example without recursion flag:
TrustedAROI example.net
```

resulting relay descriptor line containing the information when uploading the descriptor to directory authorities:
```
trusted-aroi example.com:r example.net
```

## Validating Trust Information

Trust information can be validated by verifying the relay descriptor's signature.

## Security Considerations

* Domains can expire and be registered by unrelated entities.
* Domains can be confiscated.
* Trust information and community links can also be exploited for social engeneering and other types of attacks. The design does not require anyone to link themselves to tor relay operators or reveal their identity (pseudonyms can be used).
