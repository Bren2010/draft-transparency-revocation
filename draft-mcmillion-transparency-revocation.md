---
title: "Reliable Transparency and Revocation Mechanisms"
category: info

docname: draft-mcmillion-transparency-revocation-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "Bren2010/draft-transparency-revocation"
  latest: "https://Bren2010.github.io/draft-transparency-revocation/draft-mcmillion-transparency-revocation.html"

author:
 -
    fullname: "Brendan McMillion"
    email: "brendanmcmillion@gmail.com"

normative:

informative:
  ct-in-wild:
    target: https://dl.acm.org/doi/pdf/10.1145/3319535.3345653
    title: "Certificate Transparency in the Wild: Exploring the Reliability of Monitors"
    date: 2019-11-06
    author:
      - name: Bingyu Li


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Architecture

## Requirements

The following baseline requirements for the system are as follows:


## Discussion

**Transparency requires stateful verification.** The fundamental goal of any
transparency system is to ensure that data shown to one participant is equally
visible to any other participant. This generally involves the use of a
"transparency log" that publishes data, and may or may not have additional
structure to support efficient searches for specific data. For end-users to
verify that data they've observed is properly published by the log, they must
maintain state to ensure that the views of the log they're shown are linear (in
terms of not containing forks) and internally-consistent (with respect to any
rules the log may have about its content or structure). In addition to ensuring
that the various views of the log that an end-user has seen are linear,
end-users must also *gossip* their view of the log with a wide range of other
parties. Gossiping allows end-users to detect when they've been shown a fork of
the log, which could potentially contain malicious data that other participants
in the system are unaware of.

Transparency systems that lack stateful verification, where end-users verify
that past views of the log are consistent with current views, are more
accurately termed "co-signing" schemes. That is because, with stateless
verification, the security of any such construction reduces solely to successful
verification of the transparency log's signature on a claim. The transparency
guarantees of co-signing schemes can be undermined by collusion between the
co-signers (in the web PKI, the co-signers would be a CA and one or more
transparency logs). The highly diverse and rapidly changing nature of the web
makes collusion between participants easy to achieve and difficult to detect.
This makes co-signing schemes generally unsuitable for use in the web PKI
without requiring a multitude of co-signers. However, transmitting a multitude
of signatures from larger, PQ-secure signature schemes can measurably degrade
the performance of TLS connections. Additionally, co-signing schemes provide no
way to detect collusion after-the-fact without resorting to stateful
verification.

**Servers must provide proof directly to end-users that their certificate
satisfies transparency requirements and isn't revoked.** If proof of
transparency and non-revocation isn't provided by the server, it must be fetched
from one or more third-party services. The primary issue with this is that it
ties the server's global availability to the global availability of these
third-party services, for which the server operator has no way to preempt or
resolve deficiencies. As a result of this, proposals for transparency and
revocation that rely on connectivity to third-party services have historically
been required to fail open. That is, if the third-party service is inaccessible
for too long of a period of time, the end-user stops enforcing these security
properties altogether.

A second issue is that, since we're primarily interested in describing a system
that works equally well regardless of the end-user's software vendor, these
third-party services might not be permitted to restrict access to a subset of
end-users. They would receive regular requests from all internet-connected
devices, which would present significant scaling and centralization concerns.

**Servers must refresh their certificates regularly and automatically.** This is
a direct consequence of the decision that servers must be responsible for
providing end-users proof that their certificates are not revoked. If a server
is not required to refresh its certificate, it can attempt to delay the end-user
from learning about a change of its revocation status.

**Revocation must be provided by the transparency system.** A CA can initiate
revocation either by declining to sign new statements related to a certificate
(for example, by not renewing a short-lived certificate or not creating an OCSP
staple), or by signing a new "statement of revocation" for the certificate.

In the case where CAs initiate revocation by signing a new "statement of
revocation", proving that a certificate is not revoked consists of proving that
such a statement does not exist. Proving that a statement does not exist
requires exhaustive knowledge of all such statements. Any method of conveying
this exhaustive knowledge, if it is not a transparency system, admits the
possibility of split-view attacks which are not eventually detected. Split-view
attacks in this context would allow a CA to mis-issue a certificate, claim to
revoke it, and then maintain the certificate's utility by presenting different
views of its revocation status to attack victims than to other participants. The
possibility of such a split-view attack would render revocation fundamentally
insufficient for correcting mis-issuance, and would create a need for a second
(more effective) revocation mechanism.

In the case where CAs initiate revocation by declining to sign new statements,
this makes the CA a single point-of-failure for websites relying on it. A
prolonged CA outage would have the effect of revoking all certificates and
causing a cascading outage. Proposals for revocation that fall into this
category have historically mitigated this risk by providing slower revocation,
bounded by the longest conceivable outage that a CA may have (typically at least
one week).

When specifically considering short-lived certificates as an approach to
revocation, effectiveness depends on whether or not **all** certificates are
required to be short-lived. If end-users enforce that all certificates are
short-lived, and issuance is transparent, then revocation is provided by the
transparency system as claimed. If certificates may be issued with longer
lifespans, then a second revocation mechanism for these certificates is
necessary. Considering solutions other than short-lived certificates, where the
CA initiates revocation by declining to sign some statement, it's clear that the
same potential for split-view attacks exists as discussed above.

**Transparency logs must implement the Key Transparency protocol.** As stated at
the beginning of this section, the goal of any transparency system is to ensure
that data shown to one participant is equally visible to any other participant.
An important aspect of this requirement is that server operators must be able to
contact a transparency log and verifiably receive all of the certificates and
revocations that are relevant to them.

Most transparency systems require downloading the entirety of the log's contents
to ensure that all potentially relevant entries are found. This quickly becomes
prohibitively expensive for all parties. Currently roughly 7.5 million
certificates are issued per day, with an average size of 3 kilobytes. This means
that a site operator would need to download almost 700 gigabytes of certificates
to cover a single month of issuance. Outbound bandwidth typically costs between
1 to 9 cents per gigabyte, which means that providing this data to a single
server operator would cost the transparency log between $6 to $60. For any
reasonable estimate of the number of server operators on the internet, this
represents an exceptional burden on a transparency log.

In previously deployed systems, because of this exceptional cost, site operators
have elected not to do this work themselves and instead outsourced it to
third-party monitors. Third-party monitors represent a problematic break in the
security guarantees of the system, as there are no enforceable requirements on
their behavior. They are not audited for correct behavior like certificate
authorities are, and there are no technical mechanisms to prevent misbehavior
like a transparency log would have. This has had the real-world impact of
undermining the claimed transparency guarantees of these systems {{ct-in-wild}}.

Key Transparency {{!I-D.draft-keytrans-mcmillion-protocol}} augments a
transparency log with additional structure to allow efficient and verifiable
searches for specific data. This allows server operators to download only the
contents of the transparency log that's relevant to them, while still being able
to guarantee that no certificates have been issued for their domains that they
are unaware of. As a result of the significantly reduced need for outbound
bandwidth, operating such a transparency log would cost around one million times
less than it would otherwise.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
