
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
Detecting all malicious tor network capacity in realtime is not practically feasible using active scanners and some less understood malicious relay operators
also run non-exit relays. Therefore we propose to publish relay operator trust information to subsequently have the possibility to limit the fraction
and impact of unknown tor network capacity by consuming trust information.

Trust in the context of this document is a link between community members. Trust information is unidirectional.
For example a local hackerspace might trust its community members to run tor relays without malicious intent.

additional context: [Towards a more Trustworthy Tor Network](https://media.ccc.de/v/rc3-2021-chaosstudiohamburg-475-towards-a-more-trustworthy-tor-network) (rC3-2021)

## Scope of this Document

This document describes how trust information is published, can be discovered, retrieved and validated.
It does not introduce any new requirements for tor relay operators. 

## Goal

The high level goals are

* to increase the cost for a malicious tor relay operator of getting detected 
* to increase the trustworthiness of the tor network for tor users
* to limit the impact of malicious tor relays on users
* to create incentives in the relay community to establish trust relationships

To achieve that goal we need information about whether an operator is
known by some in the community or completely unknown (no links to known entities). To some extend this is a reputation system.

On a more technical level the goal is to establish a simple protocol to publish and consume/discover trust information between relay operators.
Relay operators are identified by their authenticated relay operator IDs ([AROI](https://nusenu.github.io/OrNetStats/#authenticated-relay-operator-ids)).
As of May 2023 over [65%](https://nusenu.github.io/OrNetStats/#exit-fraction-by-aroi-over-time)
of the tor exit capacity has adopted the AROI and it is the only available specified tor relay operator identifier
that can not be arbitrarily spoofed and can be verified automatically without requiring interaction (like clicking on a link).

Trust is assigned to relay operator IDs (AROIs) and not individual relay fingerprints for scalability reasons and
so new or changed relays for a known operator get the same level of trust
without requiring any update to published trust information.

The trust scheme supports any relay type (guards, middle and exit relays).

## Design goals

- low entry barrier for publishing trust relationships
- simplicity: trust information is published by setting a torrc option or by creating one or more DNSSEC-signed DNS TXT records
- scalability: The design should support at least 10 000 AROIs
- trust relationships are public by design
- trust information must be machine readable

## Threat Model

### Adversary Goals

Adversaries want to 

* have no negative impact on future malicious relay setups even when they get detected and removed
* run their attacks without getting detected
* avoid sticking out
* maximize their network fraction of the overall tor network capacity

If they get detected they aim at having no increased setup cost when compared to the previous setup.

### Adversary Capabilities

* They can add tor relays at significant scale and broad AS diversity
* They can setup an AROI for their tor relays
* They can NOT modify another relay operator's torrc file
* They can NOT manipulate another relay operator's AROI [proof file](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md#well-knowntor-relayrsa-fingerprinttxt)
* They can NOT create valid DNSSEC-signed DNS records at a domain not under their control.

### Trust Information Consumer Goals

Trust information consumer want to learn about trusted AROIs, discover trust paths between AROIs 
and detect manipulated trust information. 
They want to follow trust relationships from tor directory authorities to individual tor relays.

## Roles

### Tor Directory Authorities (DA)

Directory authorities verify AROI proofs and publish their results in the consensus.
They also publish trusted AROIs in the consensus.
By publishing an AROI a DA asserts that they trust the operator (AROI) to run tor relays without malicious intent.
They also include information about the recursiveness of this trust relationship. So consumers know whether to follow that trust
recursively.

Directory authorities are also consumers of trust information published by relay operators.

### Relay Operators

Relay operators are identified by their [AROI](https://nusenu.github.io/OrNetStats/#authenticated-relay-operator-ids).

Relay operators can publish trust information and trust information consumers can make use of it to find relationships between relay operators. 
Trust information published by relay operators has the same meaning (asserts that the published operators are running relays without malicious intent)
as trust information published by directory authorities.

There are two ways to publish trust information for relay operators:

* via a torrc option: This easy way does not require a DNSSEC signed domain, but a relay compromise has an impact on published trust information.
* via DNSSEC signed TXT records: A relay compromise has no impact on published trust information.

If DNSSEC signed TXT records are published by an AROI, trust information published via a relay descriptor MUST be ignored to avoid downgrade attacks.

An operator can specify trust information via a relay's torrc configuration file option.
Trust information is not required to be specified in all the operator's relays. A single relay per relay operator relay set
is used to publish trust information. If multiple running relays from a given AROI publish trust information only the trust information that
was published first is taken into account. The goal here is to limit the impact of a single relay compromise. So instead of compromising an arbitrary relay
from an operator, the attacker has to comprise a specific relay.

## Publishing Trusted AROIs

Relay operators and directory authorities can publish trusted relay operator IDs (AROIs) via a single torrc option: `TrustedAROI`

Relay operators can also use DNSSEC signed TXT records. Directory authorities can NOT use DNS records.

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
### Publishing Trusted AROIs via DNS

This method can only be used by relay operators, but it is considered more secure than using the torrc `TrustedAROI` method.

The DNS labels "trusted-arois._tor." are prepended to the AROI domain to construct the DNS TXT record.
The TXT value matches the one used in the torrc option. The results from multiple DNS TXT records are merged.

```
trusted-arois._tor.example.com TXT value: "example.com:r example.net"
```

"example.com" must be the relay operator's AROI domain.


## Validating Trust Information

Trust information must be validated by verifying the relay descriptor's signature or pass the DNSSEC validation for DNS TXT records.

## Security Considerations

* Domains can expire and be registered by unrelated entities.
* Domains can be confiscated.
* Trust information and community links can also be exploited for social engeneering and other types of attacks. The design does not require anyone to link themselves to tor relay operators or reveal their identity (pseudonyms can be used).
