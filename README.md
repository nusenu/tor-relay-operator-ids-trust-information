# A Simple Web of Trust for Authenticated Relay Operator IDs (AROIs)

This is a work-in-progress draft, significant portions might change in a non-compatible way.

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
* https://gitlab.torproject.org/tpo/community/relays/-/issues/55
* [https://gitlab.torproject.org/tpo/network-health/metrics/relay-search/-/issues/40001](https://gitlab.torproject.org/tpo/network-health/metrics/relay-search/-/issues/40001)
* [https://lists.torproject.org/pipermail/tor-relays/2020-July/018656.html](https://lists.torproject.org/pipermail/tor-relays/2020-July/018656.html)

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

- simplicity: No additional PKI certificates, PGP keys or manual signing steps are required. 
Simple text files and DNS records are used.
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
* They can NOT get a valid TLS certificate for another relay operator's domain.
* They can NOT manipulate proof files 
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md#well-knowntor-relayrsa-fingerprinttxt)
by other operators.
* They can NOT create valid DNSSEC-signed DNS records at a domain not under their control.
* They CAN manipulate text files on a trust anchor's webserver.

### Trust Information Consumer Goals

Trust information consumer want to learn about trusted operator IDs and to detect manipulated trust information (content of `trusted-aroi.txt` files).

## Roles

### Trust Anchor (TA)

A trust anchor is the initial starting point which is used to find trusted AROIs.
TAs publish trusted AROIs as simple text files via HTTPS under a well-known URI. TAs can be relay operators but that is not a requirement.
TAs are identified by a DNSSEC signed DNS domain.

By publishing an AROI a TA asserts that they trust the operator - identified by their AROI - to run tor relays without malicious intent. 

Trust is binary. There is no notion of "some" trust.
Consumers of trust information can use one or more trust anchors to find trusted relay operators (identified by their AROI).

Trust anchors publish trusted AROIs via a well-known URL for trust information consumers.
Trust anchors must be able to serve a text file via HTTPS from the DNSSEC-enabled domain that trust information consumers have configured.
Additionally TAs must publish integrity information (a hash of the text file) in DNSSEC-signed TXT records. More details follow in the section "Publishing Trusted AROIs".
This allows the TA to publish trust information via semi-trusted systems (i.e. CDNs) without giving them the power to modify trust information.

### Relay Operators

Relay operators are identified by their AROI
[[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md).

Relay operators can also publish trust information and trust information consumers can make use of it to find relationships between relay operators. 
Trust information published by relay operators has the same meaning (asserts that the published operators are running relays without malicious intent) 
and requirements (HTTPS, hash published via DNSSEC-signed record).

Relay operators that do not publish trust information do not have a DNSSEC requirement on their domain.

Relay operators may also publish a reverse trust reference in a `trusted-by.txt` file to allow interested
parties to discover trusting entities. For details see the section "Publishing Trusting Parties".

### Trust Information Consumers

Consumers of trust information can choose what trust anchor they trust to find trusted AROIs.
In addition to specify the trust anchor, consumers can 
configure a global and per-TA max_depth value. The per-TA value overrides the global value. 
The max_depth describes the longest path, measured in edge counts that a consumer is willing to accept.
A consumer does not fetch and validate AROIs where max_depth is already exceeded.

| max_depth value | meaning | 
| ---            | ---     |
| -  | use the global max_depth configuration |
| -1 | accept arbitrarily long trust paths (no restrictions). |
| 0 | The TA is a trusted operator ID, no dynamic discovery of additional trusted operator IDs. No DNSSEC check is performed. |
| 1 | Discover and trust AROIs from the locally configured TA(s) but do not attempt to discover and trust any further. | 
| 2 | Discover and accept trust information with a max edge count of up to 2. In the following trust path: A(TA) -> B -> C -> D the last accepted AROI would be "C". "D" is ignored.  | 
| N | Trust up to N edges in the trust path. Ignore AROIs where the edge count exceeds this value. |

#### Trust Consumer Configuration

The max_depth value is placed after the trust anchors domain name/AROI.

Example of a consumer configuration file (ta.conf) using 3 distinct TAs:
```
global_max_depth:0
example.com:2
example.net:1
example.org:-
```

In addition to max_depth a consumer might also be interested to require multiple independend trust paths to a single AROI. 
We leave this to a future iterations of the protocol to keep it simple for now. It will not require any change on how trust information 
is published, should this be added in the future.

#### Negative Trust Configuration

A consumer can also specify a list of domains that a consumer never wants to trust for anything (no transitive trust and no relay operator trust)
to ensure that dynamic discovery will never result in any trust in the listed entities and others that are **only** discoverable via these domains.
If a certain AROI is published by a negative trust entry and by a trusted TA, the AROI is trusted.

The `negative-trust.conf` configuration contains one entry per line. Lines starting with "#" are comments and ignored during parsing.

```
malicious-TA.example.com
malicious-operator.example.com
```


## Publishing Trusted AROIs

Trust anchors and optionally also relay operators publish 
trusted relay operator IDs under this well-known URL:

https://example.com/.well-known/tor-relay/trust/trusted-aroi.txt

The URL MUST be HTTPS and use a valid certificate from a generally trusted root CA. 
Plain HTTP MUST not be used. The URL MUST be accessible by robots (no CAPTCHAs or similar).
The URL MUST NOT redirect to another host.

The file `trusted-aroi.txt` contains one AROI (a domain) per line followed by a ":" and the recursion flag: a value of 0 or 1.
This recursion flag tells the consumer whether the given AROI is also trusted to publish (1) `trusted-aroi.txt` files itself or not (0).
The recursion flag should only be enabled when the operator is known to publish `trusted-aroi.txt` and have a DNSSEC-signed domain, to avoid unnecessary polling.

Publishers MUST only publish AROIs where they are confident that the operator does NOT operate relays with malicious intent.
It is generally expected that a publisher "knows" the operators for which it vouches.

Lines starting with "#" are comments and ignored.


Example:
```
# this is a comment line
example.com:1
example.net:0
```

In addition to serving the file content via HTTPS the content MUST be 
authenticated using the following DNS TXT record - which MUST be DNSSEC-signed (DNSSEC signature MUST be validated):

```
trusted-aroi-hash._tor.example.com. TXT sha512=ee2cdf110893ad62b2aff24e668e4f8d2...
```

`trusted-aroi.txt` files that can not be verified using the hash in the DNS TXT record MUST be ignored.

The DNS TTL value for the TXT record should not exceed 60 seconds to be able to update the `trusted-aroi.txt` 
file withouth running out of sync for an extended amount of time due to DNS caching.

The only supported hash algorithm is SHA512.


## Publishing Trusting Parties

To allow interested parties to discover and enumerate trusting parties of a given AROI,
relay operators can optionally publish trusting entities under the follwoing well-known URI:

https://example.com/.well-known/tor-relay/trust/trusted-by.txt

`trusted-by.txt` contains one DNS domain by line and should not contain more than 100 entries.
`trusted-by.txt` entries can be verified by fetching the `trusted-aroi.txt` file from the well-known URI and the domains given in the `trusted-by.txt` file.
Before attempting to fetch `trusted-aroi.txt` from the domains listed in `trusted-by.txt` via HTTPS validating parties should check the DNSSEC status of the domain and the presence of the `trusted-aroi-hash._tor.example.com.` DNS TXT record to limit unnecessary HTTPS requests.

The `trusted-by.txt` file is not protected using a hash found in a DNSSEC signed TXT record, like `trusted-aroi.txt`.

## Validating Trust Information

Trust information consumers perform the following steps to find and validate trust information.

- retrieve trust anchors from local configuration
- verify they are valid DNSSEC-signed domains (not in case of max_depth=0)
- fetch AROIs from the trust anchor via HTTPS (not in case of max_depth=0)
- validate the content by fetching the hash from the related DNS TXT record (and ensure it has a valid DNSSEC signature)
- add the newly learned AROI to the local cache of trusted AROIs

Example walkthrough:

For this example walkthrough, the local configuration on the trust information consumer contains a single trust anchor:
```
example.com
```

A trust information consumer performs these steps:

1. fetch https://example.com/.well-known/tor-relay/trust/trusted-aroi.txt and ensure that the certificate is valid.
1. query the DNS TXT record `trusted-aroi-hash._tor.example.com` and ensure it is DNSSEC-signed.
1. ensure the SHA512 hash of the `trusted-aroi.txt` file matches the one provided in the DNS TXT record (from step 2)
the `trusted-aroi.txt` file contains:
```
example.org:1
example.net:0
```
1. check whether these IDs are already in the local cache of trusted operators
1. ensure example.org and example.net exist
1. add them to the list of trusted operator IDs
1. repeat the first 3 steps for example.org (but not for example.net because it has the recursion flag set to 0)

### Caching and Re-validation


Local caches at trust information consumers should not exceed 7 days.
Re-validation should happen after 4 days and MUST not occur more than once a day.


## Security Considerations

* Domains can expire and be registered by unrelated entities.
* Domains can be confiscated.
* Trust anchors can hand out distinct answers to different consumers to attack consumers. A public append only log could ensure that is detectable (similar to certificate transparency).
* Trust information and community links can also be exploited for social engeneering and other types of attacks. The design does not require anyone to link themselves to tor relay operators or reveal their identity (pseudonyms can be used).
* Depending on TA diversity the design might limits the "social diversity" of trusted AROIs to some extend.



