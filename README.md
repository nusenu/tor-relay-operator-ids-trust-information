# A Simple Web of Trust for Tor Relay Operator IDs

## Motivation

Tor users are facing persistent malicious actors
repeatedly running large fractions of the tor network's capacity to exploit them 
[[1]](https://nusenu.medium.com/the-growing-problem-of-malicious-relays-on-the-tor-network-2f14198af548)
[[2]](https://nusenu.medium.com/how-malicious-tor-relays-are-exploiting-users-in-2020-part-i-1097575c0cac)
[[3]](https://nusenu.medium.com/tracking-one-year-of-malicious-tor-exit-relay-activities-part-ii-85c80875c5df).
Detecting all malicious tor network capacity is not practically feasible using active scanners
in many cases since attackers have moved from attacking all connections to more targeted approaches where only
users of specific domains (that are not necessarily known to defenders) are exploited.
Transport level encryption (HTTPS) can defeat many types of attacks by malicious exit relays
and the global HTTPS availability has significantly increased over the past years but is still not ubiquitous yet,
especially on the first connection.
Therefore we propose to publish relay operator trust information to limit the fraction and impact of malicious tor network capacity.

Trust in the context of this document is a proxy for a link between community members.
For example a local hackerspace might trust its community members to run tor relays without malicious intent.

additional context:
* [https://gitlab.torproject.org/tpo/network-health/metrics/relay-search/-/issues/40001](https://gitlab.torproject.org/tpo/network-health/metrics/relay-search/-/issues/40001)
* [https://lists.torproject.org/pipermail/tor-relays/2020-July/018656.html](https://lists.torproject.org/pipermail/tor-relays/2020-July/018656.html)

## Scope of this Document

This document describes how trust information is published, can be retrieved and validated.
It does not introduce any new requirements for tor relay operators. 

## Goal

The high level goal is to limit the impact of malicious tor relays on users
and to increase the trustworthiness of the tor network for tor users. To achieve that goal we need
information about whether a operator is known/trusted or completely unknown (no links to known entities).

On a more technical level the goal is to establish a simple protocol to publish and consume trust information about relay operator IDs.
Relay operator IDs are the proven operator domain [[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md). 
About 60% [[6]](https://nusenu.github.io/OrNetStats/#top-10-exit-operators-with-a-proven-domain) 
of the tor exit capacity has adopted the proven operator domain as of 2021-10-01
and it is the only available specified operator identifier that can not be arbitrarily spoofed
and can automatically be verified.

Trust is issued to relay operator IDs and not individual relay fingerprints for scalability reasons and
so new or changed relays for a known operator get the same level of trust
without requiring any update to published trust information.

The trust scheme supports any relay type (guards, middle and exit relays)
but it is expected that it will be more useful for relay types that impose a higher risk
for tor users: guard and exit relays.

## Design goals

- simplicity: No additional PKI certificates, PGP keys or manual signing steps are required. 
Simple text files and DNS records are used.
- low entry barrier for publishing trusted operator IDs
- scalability: The design should support at least 10 000 operator IDs.
- trust relationships are public by design
- trust information must be machine readable
- bridges are out of scope and not supported

## Threat Model

### Adversary Goals

Adversaries want to maximize their network fraction of the overall tor network capacity
and run their attacks without getting detected.

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

Trust information consumer want to learn about trusted operator IDs and to detect manipulated trust information (content of `operator-ids.txt` files).

## Roles

### Trust Anchor (TA)

A trust anchor is the initial starting point which is used to find
trusted relay operator IDs.
Consumers of trust information can use one or more trust anchors.

All entities that publish trust information should publish their trust requirements
under this well-known URL:

https://example.com/.well-known/tor-relay/trust/requirements.txt

This file contains the rules they apply before they add a new entry to the list of trusted operator IDs in english.

Trust anchors publish trusted relay operator IDs via a well-known URL for trust information consumers.
Trust anchors must be able to serve a txt file via HTTPS from the DNSSEC enabled domain that trust information consumers have configured.
Additionally they publish integrity information in DNSSEC-signed TXT records.

Relays operated by TAs are also considered trusted if their proven operator domain matches the one from the TA.


### Relay Operators

Relay operators are identified by their proven operator domain
[[4]](https://nusenu.github.io/ContactInfo-Information-Sharing-Specification/#proof)
[[5]](https://gitlab.torproject.org/tpo/core/torspec/-/blob/main/proposals/326-tor-relay-well-known-uri-rfc8615.md).
Relay operators can also publish trust information and trust information consumers will use it when
TAs assign the operator ID the necessary recursion flag.


### Trust Information Consumers

Consumers of trust information can choose what trust anchor they trust to find trusted operator IDs.
In practice we assume that a initial default list of TAs is established. Using a common TA list also
maximizes the anonymity set size of trust information consumers.
In addition to specify the trust anchor, consumers can 
configure a global and per-TA max_depth value. The per-TA value overrides the global value. 
The max_depth describes the longest path, measured in edge counts that a consumer is willing to accept.
A consumer does not fetch and validate operator IDs where max_depth is already exceeded.

| max_depth value | meaning | 
| ---            | ---     |
| -  | use the global max_depth configuration |
| -1 | accept arbitrarily long trust paths (no restrictions). |
| 0 | The TA is a trusted operator ID, no dynamic discovery of additional trusted operator IDs. No DNSSEC check is performed. |
| 1 | Discover and trust operator IDs from the locally configured TA(s) but do not attempt to discover and trust any further. | 
| 2 | Discover and accept trust information with a max edge count of up to 2. This is the recommended default value. In the following trust path: A(TA) -> B -> C -> D the last accepted ID would be "C". "D" is ignored.  | 
| N | Trust up to N edges in the trust path. Ignore operator IDs where the edge count exceeds this value. |

**local consumer configuration**

The max_depth value is placed after the the trust anchors domain.

Example local trust anchor configuration file:
```
example.com:2
example.net:0
example.com:-
```

In addition to max_depth a consumer might also be interested to require multiple independend trust paths to a single operator ID. 
We leave this to a future iterations of the protocol to keep it simple for now. It will not require any change on how trust information is published, should this be added in the future.

## Publishing Trusted Operator IDs

Trust anchors and optionally also operators publish 
trusted relay operator IDs under this well-known URL:

https://example.com/.well-known/tor-relay/trust/operator-ids.txt

The URL MUST be HTTPS and use a valid certificate from a generally trusted root CA. 
Plain HTTP MUST not be used. The URL MUST be accessible by robots (no CAPTCHAs or similar).

The file `operator-ids.txt` contains one operator ID (a domain) per line followed by a ":" and the recursion flag: a value of 0 or 1.
This recursion flag tells the consumer whether the given operator ID is also trusted to publish (1) `operator-ids.txt` files itself or not (0).
The recursion flag should only be enabled when the operator is known to publish `operator-ids.txt` and have a DNSSEC-signed domain, to avoid unnecessary attempts.

Lines starting with "#" are comments and ignored.


Example:
```
# this is a comment line
example.com:1
example.net:0
```

In addition to serving the file content via HTTPS the content MUST be 
authenticated using the following DNS TXT record - which MUST be DNSSEC-signed:

```
operator-ids-hash._tor.example.com. TXT sha512=ee2cdf110893ad62b2aff24e668e4f8d2...
```

`operator-ids.txt` files that can not be verified using the hash in the DNS TXT record MUST be ignored.

The DNS TTL value for the TXT record should not exceed 60 seconds to be able to update the `operator-ids.txt` 
file withouth running out of sync for an extended amount of time due to DNS caching.

The only supported hash algorithm is SHA512.

## Validating Trust Information

Trust information consumers perform the following steps to find and validate trust information.

- retrieve trust anchors from local configuration
- verify they are valid DNSSEC-signed domains (not in case of max_depth=0)
- fetch operator IDs from the trust anchor via HTTPS (not in case of max_depth=0)
- validate the content by fetching the hash from the related DNS TXT record (and ensure it is DNSSEC-signed)
- add the newly learned operator IDs to the local cache of trusted operator IDs

Example walkthrough:

For this example walkthrough, the local configuration on the trust information consumer contains a single trust anchor:
```
example.com
```

A trust information consumer performs these steps:

1. fetch https://example.com/.well-known/tor-relay/trust/operator-ids.txt and ensure that the certificate is valid.
1. query the DNS TXT record `operator-ids-hash._tor.example.com` and ensure it is DNSSEC-signed.
1. ensure the SHA512 hash of the `operator-ids.txt` file matches the one provided in the DNS TXT record (from step 2)
the `operator-ids.txt` file contains:
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
* Trust anchors can hand out distinct answers to different consumers to attack trust information consumers. A public append only log could ensure that is detectable (similar to certificate transparency).



