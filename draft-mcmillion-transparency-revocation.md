
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
views of its revocation status to attack victims than to other participants. In
essence, the possibility of such a split-view attack renders revocation an
ineffective tool for correcting mis-issuance.

In the case where CAs initiate revocation by declining to sign new statements,
this makes the CA a single point-of-failure for websites relying on it. A
prolonged CA outage would have the effect of revoking all certificates and
causing a cascading outage. Proposals for revocation that fall into this
category have historically mitigated this risk by providing slower revocation,
bounded by the longest conceivable outage that a CA may have (typically at least
one week).

When specifically considering short-lived certificates as an approach to
revocation, their effectiveness relies on whether or not **all** certificates
are required to be short-lived. If end-users enforce that all certificates are
short-lived, and issuance is transparent, then revocation is provided by the
transparency system. If certificates may be issued with longer lifespans, then a
secondary revocation mechanism for these certificates is necessary. Considering
solutions other than short-lived certificates, where the CA initiates revocation
by declining to sign some statement, it's clear that the same potential for
split-view attacks exists as discussed above.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
